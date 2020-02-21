## AKS Scan With Alcide Advisor

> Codelab with additional usecases & detection can be [here](https://codelab.alcide.io/codelabs/advisor-codelab-01/index.html#0)

```sh
mkdir -p /tmp/training/advisor
```

#### For Linux
``` sh
cd /tmp/training/advisor &&\
curl -o advisor https://alcide.blob.core.windows.net/generic/stable/linux/advisor &&\
chmod +x advisor
``` 

#### For Mac 
``` sh
cd /tmp/training/advisor &&\
curl -o advisor https://alcide.blob.core.windows.net/generic/stable/darwin/advisor &&\
chmod +x advisor
```

#### Scan Your AKS Cluster

We are going to start with an initial cluster scan using the buitin scan profile.


``` bash
cd /tmp/training/advisor &&\
 ./advisor validate cluster --cluster-context <your-aks-context-name> \
--namespace-include="*" --namespace-exclude="-" --outfile scan.html
```

Open in your browser the generated report **scan.html** and review the result across the various categories.

``` bash
google-chrome scan.html&   or   open scan.html&
```

Review the results