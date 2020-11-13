---
title: "Ad-hoc certbot on Kubernetes"
date: 2020-11-12T17:51:45-08:00
tags: [ kubernetes, docker, bash ]
draft: true
---
I've been working on building up my first web page using Kubernetes and I ran into a snag pretty early on using Certbot. Certbot needs to be able to be behind my NGINX ingress, which means deploying it, but I can't actually have it handle automatically installing the certs. There are some out of the box solutions like the [jetstack/cert-manager operator](https://github.com/jetstack/cert-manager). I was a little iffy on trusting my certs to a black box solution, and given the small scale of my cluster (single node, two hostnames) I just wanted a quick and easy method.

My first attempt was to just use the default certbot docker and try putting it behind an ingress. Certbot expects to be run behind nginx so I had to fiddle a bit with the commands that I was passing to the entry point. I ended up with something this:

    docker run certbot/certbot certonly --standalone --non-interactive --agree-tos -m cebolladelanoche@gmail.com -d pretzy.xyz

This worked fine except for one problem: retrieving the certs. Already my "simple" solution was becoming more complicated. I could do a volume mount but this felt like overkill. Even if I did a volume mount I'd still have to remount it later to retrieve the certs. I really only wanted the container to exist long enough to validate the letsencrypt challenge and generate the certificates. I just needed it to stay alive long enough for me to shell in and grab them, however after certbot runs it immediately stops. I wound up just creating a simple dockerfile to run the bot and then sleep long enough for me to grab the goods:

dockerfile

    FROM certbot/certbot:latest
    COPY ./run.sh /etc/run.sh
    ENTRYPOINT ["/bin/sh", "/etc/run.sh"]

run.sh

    #!/bin/sh
    /usr/local/bin/certbot certonly --standalone --non-interactive --agree-tos -m cebolladelanoche@gmail.com -d ${URL}
    sleep 1000

In retrospect, I believe I could have just used the --deploy-hook with the vanilla image to achieve a similar result. After checking the logs to ensure that the certbot had run successfully I ran ```kubectl exec -it deployment/certbot -- /bin/sh```. I did a quick cat of the certs and saved them locally to be added into kubernetes. Obviously not a very repeatable process, but it got the job done. This was fine for the first domain, but when I added a second domain followed by a subdomain I realized that I really needed an easier solution. Do I dare use the easy solution? Nah, why not hack together some home-grown method!

certbot.sh

    #!/bin/bash

    #Ensure that a domain is specified
    if [ -z $1 ];
    then
            echo "Missing required value"
            exit 1
    fi;

    export URL=${1}

    #build and push, just in case
    docker build certbot -t iad.ocir.io/idcwcdqc5wko/pretzy/certbot
    docker push iad.ocir.io/idcwcdqc5wko/pretzy/certbot

    #substitute in the URL to my kubernetes configs
    cat certbot.yml | envsubst | kubectl apply -f -
    echo "Waiting for the pod to create..."
    sleep 30

    #find the pod. Kubernetes doesn't seem to like using cp for a deployment
    POD=$(kubectl get pods -l app=certbot -o json | jq -r '.items[0].metadata.name')

    #dump those certs!
    for f in cert1 chain1 fullchain1 secret1;
    do
        kubectl cp ${POD}:etc/letsencrypt/archive/${URL}/${f}.pem ./certs/${URL}.${f}.pem
    done

    #display any errors
    if [[ $! != 0 ]];
    then
            echo "Problem copying certs. Pulling up logs before deleting."
            kubectl logs deployment/certbot
    fi;

    #clean up
    cat certbot.yml | envsubst | kubectl delete -f -

certbot.yml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: certbot
      namespace: default
    spec:
      selector:
        matchLabels:
          app: certbot
      template:
        metadata:
          labels:
            app: certbot
        spec:
          imagePullSecrets:
          - name: ocirsecret
          containers:
          - name: certbot
            image: internal.registry/certbot:latest
            resources:
              limits:
                memory: "128Mi"
                cpu: "500m"
            ports:
            - containerPort: 80
            - containerPort: 443
            env:
              - name: URL
                value: ${URL}

    ---

    apiVersion: v1
    kind: Service
    metadata:
      name: certbot
    spec:
      selector:
        app: certbot
      ports:
      - name: http
        port: 80
        targetPort: 80
      - name: https
        port: 443
        targetPort: 443

    ---

    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: certbot
      namespace: default 
      annotations:
        kubernetes.io/ingress.class: "nginx"
    spec:
      rules:
      - host: ${URL}
        http:
          paths:
          - path: /
            backend:
              serviceName: certbot
              servicePort: 80


So this solution wound up working pretty well for my purposes. It also makes it easy to generate certs that you don't plan on using inside kubernetes, ie if you're hosting some of your subdomains outside of kubernetes. I think for most people cert-manager is probably the best solution and it's likely that I'll move to using that at some point in the future. I feel like this setup still has some limited value for anyone looking for a quick solution to generating letsencrypt certs on Kubernetes.