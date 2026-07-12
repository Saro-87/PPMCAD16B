---

GitOps: Git should be your single source of truth for everything, your code, you infrastructure code ...

Pre-Req: the infrastructure should be created via code and the code should be kept in a git repo..

e.g.:

- anything in "main/release" branch should be deployable to "production" env
- anything in "qa" branch should be deployable to "qa" env
- anything in "develop / integration" branch should be deployable to "dev" env

ArgoCD and another is FluxCD

---

ArgoCD: 

- You install ArgoCD in your Kubernetes cluster... 5/6 components of ArgoCD gets deployed on the cluster... git watcher, redis, authentication etc..

- you create one file, named as application.yaml where in you mention which git repo and which branch and folder inside that repo, argo should continuously watch..



-------------------

Q.) Devs should not put .env file in the git repo as the file contains secrets, credentials etc. but how do anyone who is using your git repo know that these are the variable they should set before running the code else the code will fail?

A.) README.md file, I need to setup payment_api_key in order to run the code.., .env.example without any real value.. mysql_host, mysql_user, mysql_pass, and in AWS you can create this via RDS and fetch the credential by following xyz steps..

-------------------