# test-helm-repo


``` bash
# helm repo add myhelm1 https://hfbhfb.github.io/test-helm-repo/ # 加入
# helm search repo myhelm1 -l


hfbdeMacBook-Pro:test-helm-repo hfb$ helm create ccipack
hfbdeMacBook-Pro:test-helm-repo hfb$ cd ccipack/
hfbdeMacBook-Pro:ccipack hfb$ ls
Chart.yaml  charts      templates   values.yaml
hfbdeMacBook-Pro:test-helm-repo hfb$ helm package ccipack
Successfully packaged chart and saved it to: /Users/hfb/test/test-helm-repo/ccipack-0.1.0.tgz
hfbdeMacBook-Pro:test-helm-repo hfb$ helm repo index .
#git push origin gh-pages

```

