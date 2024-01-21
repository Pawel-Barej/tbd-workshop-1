IMPORTANT ❗ ❗ ❗ Please remember to destroy all the resources after each work session. You can recreate infrastructure by creating new PR and merging it to master.

![img.png](doc/figures/destroy.png)

0. The goal of this phase is to create infrastructure, perform benchmarking/scalability tests of sample three-tier lakehouse solution and analyze the results using:
* [TPC-DI benchmark](https://www.tpc.org/tpcdi/)
* [dbt - data transformation tool](https://www.getdbt.com/)
* [GCP Composer - managed Apache Airflow](https://cloud.google.com/composer?hl=pl)
* [GCP Dataproc - managed Apache Spark](https://spark.apache.org/)
* [GCP Vertex AI Workbench - managed JupyterLab](https://cloud.google.com/vertex-ai-notebooks?hl=pl)

Worth to read:
* https://docs.getdbt.com/docs/introduction
* https://airflow.apache.org/docs/apache-airflow/stable/index.html
* https://spark.apache.org/docs/latest/api/python/index.html
* https://medium.com/snowflake/loading-the-tpc-di-benchmark-dataset-into-snowflake-96011e2c26cf
* https://www.databricks.com/blog/2023/04/14/how-we-performed-etl-one-billion-records-under-1-delta-live-tables.html

2. Authors:

   ***Enter your group nr***

   ***Link to forked repo***

3. Replace your `main.tf` (in the root module) from the phase 1 with [main.tf](https://github.com/bdg-tbd/tbd-workshop-1/blob/v1.0.36/main.tf)
and change each module `source` reference from the repo relative path to a github repo tag `v1.0.36` , e.g.:
```hcl
module "dbt_docker_image" {
  depends_on = [module.composer]
  source             = "github.com/bdg-tbd/tbd-workshop-1.git?ref=v1.0.36/modules/dbt_docker_image"
  registry_hostname  = module.gcr.registry_hostname
  registry_repo_name = coalesce(var.project_name)
  project_name       = var.project_name
  spark_version      = local.spark_version
}
```


4. Provision your infrastructure.

    a) setup Vertex AI Workbench `pyspark` kernel as described in point [8](https://github.com/bdg-tbd/tbd-workshop-1/tree/v1.0.32#project-setup) 

    b) upload [tpc-di-setup.ipynb](https://github.com/bdg-tbd/tbd-workshop-1/blob/v1.0.36/notebooks/tpc-di-setup.ipynb) to 
the running instance of your Vertex AI Workbench

5. In `tpc-di-setup.ipynb` modify cell under section ***Clone tbd-tpc-di repo***:

   a)first, fork https://github.com/mwiewior/tbd-tpc-di.git to your github organization.

   b)create new branch (e.g. 'notebook') in your fork of tbd-tpc-di and modify profiles.yaml by commenting following lines:
   ```  
        #"spark.driver.port": "30000"
        #"spark.blockManager.port": "30001"
        #"spark.driver.host": "10.11.0.5"  #FIXME: Result of the command (kubectl get nodes -o json |  jq -r '.items[0].status.addresses[0].address')
        #"spark.driver.bindAddress": "0.0.0.0"
   ```
   This lines are required to run dbt on airflow but have to be commented while running dbt in notebook.

   c)update git clone command to point to ***your fork***.

 


6. Access Vertex AI Workbench and run cell by cell notebook `tpc-di-setup.ipynb`.

    a) in the first cell of the notebook replace: `%env DATA_BUCKET=tbd-2023z-9910-data` with your data bucket.


   b) in the cell:
         ```%%bash
         mkdir -p git && cd git
         git clone https://github.com/mwiewior/tbd-tpc-di.git
         cd tbd-tpc-di
         git pull
         ```
      replace repo with your fork. Next checkout to 'notebook' branch.
   
    c) after running first cells your fork of `tbd-tpc-di` repository will be cloned into Vertex AI  enviroment (see git folder).

    d) take a look on `git/tbd-tpc-di/profiles.yaml`. This file includes Spark parameters that can be changed if you need to increase the number of executors and
  ```
   server_side_parameters:
       "spark.driver.memory": "2g"
       "spark.executor.memory": "4g"
       "spark.executor.instances": "2"
       "spark.hadoop.hive.metastore.warehouse.dir": "hdfs:///user/hive/warehouse/"
  ```


7. Explore files created by generator and describe them, including format, content, total size.

       Generator wrzucił dane do ścieżki /tmp/tpc-di. Dane były przechowywane na następujących formatach:
       .txt, .xml, .csv.
        Różne rodzaje danych w różny sposób wpływały na obciążenie.
        Struktóra danych to
         Batch1, Batch_audit.csv, Batch2, Batch2_audit.csv,  Batch3, Batch3_audit.csv, Generator_audit.csv, digen_report.txt
         Batch1 zużywa prawie 1GB danych i składa się z plików csv wykorzystywanych w bazach danych, składa sie z prawie 16mln rkordów
         Batch2 oraz Batch3 wykorzystuje tylko lekko ponad 12MB danych wykorzystywanych w póxniejszych etapach, tutaj jest tylko niecałe 70000 rekordów.

9. Analyze tpcdi.py. What happened in the loading stage?

       Jest to skrypt, który spowodował że dane przechowywane lokalnie zostały wysłane do chmurowego storage.
       Możemy wyróżnić takie fazy jak:
       1. Przetwarzanie danych w tcp-di
       2. Utworzenie storage dla danych
       3. Przesłanie danych do storage
       4. Przeslanie danych do spark dataFrames

11. Using SparkSQL answer: how many table were created in each layer?

          demo_bronze ma 17 tabel
          demo_silver ma14 tabel
          demo_gold ma 12 tabel
          digen ma 17 tabel

11. Add some 3 more [dbt tests](https://docs.getdbt.com/docs/build/tests) and explain what you are testing.

          Pierwszy test sprawdzający czy Czy każdy sk_trade jest unikatowy w tabeli.
          Wykonalismy 3 podobne testy dla inny tabl
    
          select 
          sk_trade_id, 
          count(*) cnt
          from {{ ref('fact_trade') }} 
          group by sk_trade_id
            having cnt > 1


          select 
          sk_broker_id, 
          count(*) cnt
          from {{ ref('fact_trade') }} 
          group by sk_broker_id
            having cnt > 1



              select 
          sk_customer_id, 
          count(*) cnt
          from {{ ref('fact_cash_balances') }} 
          group by sk_customer_id
            having cnt > 1


11. In main.tf update
   ```
   dbt_git_repo            = "https://github.com/mwiewior/tbd-tpc-di.git"
   dbt_git_repo_branch     = "main"
   ```
   so dbt_git_repo points to your fork of tbd-tpc-di. 

12. Redeploy infrastructure and check if the DAG finished with no errors:

![image](https://github.com/Pawel-Barej/tbd-workshop-1/assets/89931555/f3f05564-f6a1-472e-bb02-ff3033cf4335)

