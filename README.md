## Deploy do cluster K8s

- ### kind
    kind create cluster --config infra/kind/kind-cluster-config.yaml

- ### Amazon EKS
    1. Instalar o [eksctl](https://eksctl.io/introduction/)

    2. Para criar o cluster, execute:```eksctl create cluster -f infra/eks/cluster.yaml --profile <name>```
    3. Para escalabilidade com Cluster Autoscaler(CA), seguir [essa](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) documentação oficial
    4. Para escalabilidade com o Karpanter, seguir [essa](https://karpenter.sh/v0.25.0/getting-started/getting-started-with-eksctl/) documentação

## Construindo uma imagem do Spark
  Existem algumas formas de buildar uma imagem do Spark. A primeira delas e mais recomendada é criando um arquivo Dockerfile.

  Nesse repo, foi disponibilizado uma alternativa similar ao Dockerfile, que além de buildar a imagem, aplica um path do Hive necessário para integração com o Glue

  Consulte o material em ```labs/2_building-spark-image```

## Instalando Spark Operator
Criar o namespace aonde os jobs spark irão rodar 
   
    $ kubectl create namespace processing 

Instalar o operador do Spark

    $ helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator

    $ helm install spark-operator -f helm/spark-operator/values.yaml spark-operator/spark-operator --namespace spark-operator --create-namespace
  
Essa instalação, já está habilitado o Mutanting Admission Webhook

## Execução do primeiro job Spark no K8s
  Execute o comando para submeter um job de exemplo no K8s

  ```kubectl apply -f labs/3_hello-world-spark-operator ```
## Instalando Apache Airflow no K8s

    $ helm install airflow apache-airflow/airflow --namespace orchestrator --create-namespace --debug
    
    # habilita permissão para que o airflow consiga submeter o YAML do Spark
    $ helm upgrade --install airflow apache-airflow/airflow -n orchestrator -f ./helm/airflow/values.yaml --debug
    $ kubectl apply -f helm/airflow/config-permission.yaml
    
    # mapeia a porta do localhost para o container
    $ kubectl port-forward svc/airflow-webserver 8080:8080 --namespace orchestrator
    
    # Usuario: admin
    # Senha: admin
## Monitorando jobs Spark

Um dos primeiros passos para subir a stack de monitoramento é buildar uma imagem do spark com o jar do Prometheus. Nesse exemplo, estamos buildando a imagem através do arquivo [Dockerfile](labs%2Fmonitoring-spark-jobs%2FDockerfile) passando o jar [jmx_prometheus_javaagent-0.17.2.jar](labs%2Fmonitoring-spark-jobs%2Fjmx_prometheus_javaagent-0.17.2.jar)

    $ cd labs/monitoring-spark-jobs
    $ docker build -t <seu-dockerhub-registry>/spark:3.2.3-prometheus .

    # upload da imagem buildada para seu registry de imagens
    $ docker push <seu-dockerhub-registry>/spark:3.2.3-prometheus

O próximo passo é instalar o Prometheus e o Grafana via helm

### Prometheus

    $ helm install prometheus -f helm/prometheus/values.yaml helm/prometheus --namespace monitoring --create-namespace
    $ export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
    $ kubectl --namespace monitoring port-forward $POD_NAME 9090   

  
Acesse a URL: localhost:9091

### Grafana
    $ helm install grafana -f helm/grafana/values.yaml helm/grafana --namespace monitoring
    $ export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
    $ kubectl --namespace monitoring port-forward $POD_NAME 3000

Acesse a URL: localhost:3000

  * Usuário: admin
  * 
    Rode o comando para obter a senha de login no Grafana
  
    $ kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

### Testando execução com monitoramento
  
    $ kubectl apply -f labs/10_monitoring-spark-jobs/prometheus-config-map.yaml -n processing
    $ kubectl apply -f labs/10_monitoring-spark-jobs/job-prometheus-integration.yaml

Consulte o Prometheus e/ou Grafana para visualizar métricas da execução

Ao adicionar o Data Source no Grafana, espeficique a url interna do Prometheus
http://prometheus-server.monitoring.svc.cluster.local


Referências/Dicas

* [https://aws.amazon.com/blogs/containers/best-practices-for-running-spark-on-amazon-eks/](https://aws.amazon.com/blogs/containers/best-practices-for-running-spark-on-amazon-eks/)
* [https://aws.amazon.com/blogs/containers/optimizing-spark-performance-on-kubernetes/](https://aws.amazon.com/blogs/containers/optimizing-spark-performance-on-kubernetes/)
* [https://www.amazon.com/Managing-Cloud-Native-Data-Kubernetes/dp/1098111397](https://www.amazon.com/Managing-Cloud-Native-Data-Kubernetes/dp/1098111397)
* https://www.amazon.com/Managing-Cloud-Native-Data-Kubernetes/dp/1098111397
* https://dok.community/
* https://spot.io/blog/ocean-for-apache-spark-serverless-spark-now-available-on-aws/
* https://docs.datamechanics.co/docs/welcome
