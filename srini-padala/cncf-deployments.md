## Tekton Pipelines

To enable native CICD on kubernetes cluster using Tekton below are the components to be installed

- Tekton Pipelines: The core components that define CICD pipelines and run them
- Tekton Dashboard: The GUI that helps view and manage pipelines/runs
- Tekton Triggers: Enables to hook the CICD pipelines to source control events

### Install Tekton Pipelines
    kubectl apply -f https://storage.googleapis.com/tekton-releases/latest/release.yaml
Apply the configuration to prevent Tekton from overriding the working directory for the containers
    kubectl apply -f tekton-feature-flags-configmap.yaml -n  tekton-pipelines

### Install Tekton Triggers
    kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

### Install Tekton Dashboard
    kubectl apply --filename https://github.com/tektoncd/dashboard/releases/download/v0.5.2/tekton-dashboard-release.yaml

#### Create Ingress for dashboard
The ingress uses external DNS to create a CNAME record with a target. Update the hostnames or correct the ingress definitions if DNS record is created manually
    kubectl apply -f tekton-dashboard-ingress.yaml

###Configure Triggers (Refer the yamls in the yaml folder)

1. Create the roles required to run the pipelines and webhooks (update the namespace)

	    kubectl apply -f  admin-role.yaml  -n <<namespace>>
	    kubectl apply -f  webhook-role.yaml  -n <<namespace>>

2. Create Ingress Task

    The template task to create the ingress for the webhook to reach the listener. The template uses external DNS to create a CNAME record with a target. Update the hostnames or correct the ingress definitions if DNS record is created manually

	    kubectl apply -f create-ingress.yaml -n <<namespace>>

3. Create Webhook Task  - create-webhook.yaml

    The template task to create the webhook on the github repo. Template Tasks will be used by TaskRuns later.
	
	    kubectl apply -f create-webhook.yaml -n <<namespace>>

4. Create the Pipeline components and Trigger Bindings (Pipeline => pipeline + tasks)

    Create the Taks and Pipelines as required

        kubectl apply -f pipeline.yaml
	
    Update the Triggers that invokes the pipelines (Event Listener => Trigger Binding + Trigger Template)

	    kubectl apply -f triggers.yaml

5. Task Runs for create ingress and create webhook

	Create a PAT token to let the task create the webhook in github. The permission should include public_repo, admin:repo_hook. Update the token and run the below

	    kubectl apply -f github-secret.yaml -n <<namespace>>

	Update the github details and the webhook ingress

	    kubectl apply -f webhook-run.yaml -n <<namespace>>
	
	Update the ingress for the webhook, the event listener service name and run the below

	    kubectl apply -f ingress-run.yaml -n <<namespace>>

##References:

https://github.com/tektoncd/pipeline/blob/master/docs/install.md

https://github.com/tektoncd/triggers/blob/master/docs/getting-started/README.md

https://github.com/tektoncd/dashboard
 