export GITHUB_TOKEN=token

az login

az aks get-credentials --resource-group kluster1 --name aks

flux check --pre

flux bootstrap github \
--owner girishgouda \
--personal true \
--repository flux

kubectl -n flux-system get gitrepositories.source.toolkit.fluxcd.io

kubectl -n flux-system get kustomizations.kustomize.toolkit.fluxcd.io

git clone git@github.com:girishgouda/flux.git