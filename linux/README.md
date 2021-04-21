## Linux

#### Test Load
```
stress --vm "1" --vm-bytes 50M --vm-hang "1"
```

#### GitLab, working with non-defaut SSH key pair paths
```
# Ref https://docs.gitlab.com/ee/ssh/
eval $(ssh-agent -s)
ssh-add <path to private SSH key>
```

#### Add comma or any other string at the end of the line
```
sed "s/$/,/" es-open-indices-qa.txt
```

#### Use xargs to delete list of pods
```
kubectl get pod --all-namespaces -o wide | grep airflow-scheduler | awk '{print $2}' | xargs -I{} kubectl delete pod {} -n airflow
```
