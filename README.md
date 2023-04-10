Create the policy:

```
aws iam create-policy \
	--policy-name cert-manager-policy \
	--policy-document file://policy.json
```

Create the role:

```
aws iam create-role \
	--role-name cert-manager \
	--assume-role-policy-document file://trust.json
```

Update the role policy if changes are made to `.json` file

```
aws iam update-assume-role-policy \
	--role-name cert-manager \
	--policy-document file://trust.json # change to irsa.json for oidc
```

To get the oidc id from the eks cluster run:
`aws eks describe-cluster --name cluster-name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5`

Attach the role policy:

```
aws iam attach-role-policy \
	--role-name cert-manager \
	--policy-arn arn:aws:iam::${AWS_ID}:policy/cert-manager-policy
```

to get the hostedzone ID:
`aws route53 list-hosted-zones | jq -r '.HostedZones[] | select(.Name=="${DOMAIN_NAME}.com.") | .Id'`

When trying to use environment and test the policy:

```
aws iam simulate-principal-policy \
 --policy-source-arn arn:aws:iam::${AWS_ID}:role/cert-manager \
 --action-names route53:ListResourceRecordSets \
 --resource-arn arn:aws:route53:::hostedzone/${HOSTEDZONE_ID}
```

To deploy cert-manager using helm:

```
	cert-manager helm install \
		cert-manager jetstack/cert-manager \
		--namespace cert-manager \
		--create-namespace \
		--version v1.11.0 \
		--set installCRDs=true \
		--values "values.yaml" \
		--wait
```

where values:

```
	serviceAccount:
	annotations:
	eks.amazonaws.com/role-arn: arn:aws:iam::${AWS_ID}:role/cert-manager

	installCRDs: true

	# the securityContext is required, so the pod can access files required to assume the IAM role
	securityContext:
	fsGroup: 1001

	extraArgs:
	- '--dns01-recursive-nameservers-only'
	- '--dns01-recursive-nameservers=8.8.8.8:53,1.1.1.1:53'
```

Only need ClusterIssuer and Ingress resource since using ingress shim with annotations

`cert-manager.io/cluster-issuer: 'letsencrypt-prod'`

When applying yaml files, pass in environmental variables like so:

`kubectl apply -f cluster-issuer-prod.yaml --env AWS_ID=111122223333`
