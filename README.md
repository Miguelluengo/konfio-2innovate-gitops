# Ejemplo gitops

Este repositorio nos permite evidenciar cómo es el [marco de trabajo con
GitOps](https://gitops-workflow.mikroways.net/) que proponemos desde
[Mikroways](https://mikroways.net/).

La implementación aquí planteada, barrerá la carpeta projects. En esta
estructura tenemos:

* `custombank/`: es el nombre de un tenant/cliente
  * `non-prod/`: es el nombre de uno de los dos clusters de éste cliente. En
    este caso el de *non-prod*.
        * `dev/`: el ambiente. Este caso *dev*, pero puede ser *qa*, *test*,
          etc.
            * `service-1/`: el nombre de algun servicio
  * `prod/`: el nombre del cluster de prod.
    * Se repite la lógica expresada para *non-prod*.

Para que funcione esta jerarquía en el último directorio debe existir un
`values.yaml`. Este values, define lo que se explica en el chart que [estamos
configurando](charts/custom-argo-project). La intención de éste chart, es la de
crear para cada servicio mencionado, en cada cluster y para cada ambiente:

* Una Aplicación de argo en el proyecto **default**. Esta aplicación implementa
  el patrón de [Apps of Apps de ArgoCD](https://argo-cd.readthedocs.io/en/latest/operator-manual/cluster-bootstrapping/).
* Esta aplicación creará:
    * Un proyecto para el servicio en este ambiente
    * La aplicación propiamente dicha
    * Una serie de requerimientos que podría neecsitarse previo al despliegue de
      la aplicación. Este paso es opcional.

Si observamos alguno de los values de ejemplo, [como ser éste](projects/custombank/non-prod/dev/ms-accumulator/values.yaml),
podemos observar que al mezclar (merge entre el yaml) con el yaml del
ApplicationSet mencionado en el repositorio [2innovateit-pts/poc-aws-base-services](https://github.com/2innovateit-pts/poc-aws-base-services/)
obtendremos algo como:

```yaml
    application:
        enabled: true
        sources:
          - repoURL: registry.2innovateit.io
            chart: helm-charts-mvp/2innovateit-apps
            targetRevision: 0.0.27
            helm:
              values: |
                company: custombank
                env: dev
                environments:
                    A0_APP_ENV: dev
                    A0_COMPANY: custombank
              valueFiles:
                - $values/common/values.yaml
                - $values/common/dev/values.yaml
                - $values/aws/custombank/ms-accumulator/dev/values.yaml
          - repoURL: https://github.com/2innovateit-pts/pocs-2innovate-gitops-clients.git
            targetRevision: main
            ref: values
```

> Este yaml se muestra hidratado con los valores de haber aplicado el
> ApplicationSet contra el values en éste repositorio.

Significa entonces, que la aplicación de argo, creará un despliegue del chart en
una registry oci, con url
oci://registry.2innovateit.io/helm-charts-mvp/2innovateit-apps, en la versión
0.0.27, con los valores aplicados con la siguiente presedencia:

* Primero los values mostrados:
    * company
    * env
    * environments
* Luego los yaml desde:
    * El repo https://github.com/2innovateit-pts/pocs-2innovate-gitops-clients.git
    * En la rama main
    * En el orden:
        * common/values.yaml
        * common/dev/values.yaml
        * aws/custombank/ms-accumulator/dev/values.yaml
