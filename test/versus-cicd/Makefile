account = $(shell aws sts get-caller-identity --query 'Account' --output text)
app = versus
database = $(shell aws rds describe-db-instances --region us-east-1 --query 'DBInstances[*].Endpoint.Address' | grep versus |  sed 's/ //g;s/"//g')
stage = dev
include stages/$(stage)
region = us-east-1
cluster = Ubuntu-team-EKS

build-back:
	cd backend/ && docker build -t $(account).dkr.ecr.$(region).amazonaws.com/$(app)/back:$(back-version) .

build-front:
	sed -i "s/APP/$(app)/2;s/HOSTNAME/$(hostname)/g" ./frontend/.env.production
	cd frontend/ && docker build -t $(account).dkr.ecr.$(region).amazonaws.com/$(app)/front:$(front-version) .

push-back:
	aws ecr get-login-password --region $(region) | docker login --username AWS --password-stdin $(account).dkr.ecr.$(region).amazonaws.com
	docker push $(account).dkr.ecr.$(region).amazonaws.com/$(app)/back:$(back-version)

push-front:
	aws ecr get-login-password --region $(region) | docker login --username AWS --password-stdin $(account).dkr.ecr.$(region).amazonaws.com
	docker push $(account).dkr.ecr.$(region).amazonaws.com/$(app)/front:$(front-version)

create-ns:
	aws eks --region $(region) update-kubeconfig --name $(cluster)
	cat kubernetes/namespaces.yaml | sed "s/NAMESPACE/$(namespace)/g" | kubectl apply -f -

deploy: 
	cat kubernetes/rds-service.yaml | sed "s/NAMESPACE/$(namespace)/g;s/ENDPOINT/$(database)/g" | kubectl apply -f -
	cat kubernetes/backend.yaml | sed "s/IMAGEVERSION/$(back-version)/g;s/ACCOUNT/$(account)/g;s/NAMESPACE/$(namespace)/g;s/APP/$(app)/g;s/REGION/$(region)/g" | kubectl apply -f -
	cat kubernetes/frontend.yaml | sed "s/IMAGEVERSION/$(front-version)/g;s/ACCOUNT/$(account)/g;s/NAMESPACE/$(namespace)/g;s/APP/$(app)/g;s/REGION/$(region)/g" | kubectl apply -f -
	cat kubernetes/ingress-rules.yaml | sed "s/APP/$(app)/g;s/HOSTNAME/$(hostname)/g;s/NAMESPACE/$(namespace)/g" | kubectl apply -f -

create-rds-secrets:
	chmod +x secret-create.sh
	./secret-create.sh rds ./kubernetes/secrets

update-rds-secrets: 
	chmod +x secret-update.sh
	./secret-update.sh rds ./kubernetes/secrets

inject-rds-secrets:
	chmod +x secret-inject.sh
	./secret-inject.sh rds $(namespace) ./kubernetes/secrets

delete-all:
	cat kubernetes/namespaces.yaml | sed "s/NAMESPACE/$(namespace)/g" | kubectl delete -f -
