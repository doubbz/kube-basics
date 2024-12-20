# require un nouveau package

```sh
# ajouter le repo
helm repo add elastic https://helm.elastic.co

# pull le .tgz
helm pull elastic/eck-operator -d <path where to put it>

# install dans le cluster courant
helm upgrade --install eck-operator -f ./deps/values/values.eck.yaml ./deps/charts/eck-operator-2.11.1.tgz --wait --wait-for-jobs
```

Sinon on peut passer par les fichiers Chart.yaml|Chart.lock
```sh
# pré-requis = ajouter la dependence dans le Chart.yaml
# générer/mettre à jour Chart.lock + télécharger le .tgz
helm dependency update
# installer
helm upgrade --install bns --wait --wait-for-jobs -f ./apps/values/values.yaml
```

Sinon en mode quick n dirty p ex pour tester un repo helm, cette commande suffit :
```sh
helm upgrade -i grafana-operator oci://ghcr.io/grafana/helm-charts/grafana-operator --version v5.6.0
```

# Helm 2 vs Helm 3

Helm 3 est une enorme diff vs la v2 et n'est pas du tout backward compatible (les docs sont souvant peu utilisables)
Historiquement, à l'époque k8s n'avait pas de RBAC et pas de CRDs non plus donc helm 2 s'appuyait sur un composant instatllé dans le cluster i.e. Tiller. Helm 3 apporte aussi la strategie "3-way merge" qui permet de rollback des modifs manuelles ou d'upgrade sans ecraser des modifs customs car elle tient compte de l'état courant du cluster alors que helm 2 non.

# Tips

* debug la generation du manifest YAML
```sh 
helm install mychart --dry-run --debug ./mychart
```

* pour lister les values d'un repo
```sh
# installer le chart ("en local" i.e. sur le host là où est lancé la commande)
helm repo add <name> <chart>

# display all values
helm show values <name>
```

* [super useful documentation on values.yaml special cases](https://helm.sh/docs/chart_template_guide/yaml_techniques/)

* plugin Helm Diff pour plan/review la diff d'un helm upgrade avant de "l'apply"
```sh
helm plugin install https://github.com/databus23/helm-diff
```

* in order to dump all CustomResourceDefinition from a given helm release
```sh
helm template bns ./helm -f helm/values.dev.yaml | awk '/^---$/ {if (content ~ /CustomResourceDefinition/) {print content; print "---"} content=""; next} {content=content"\n"$0} END {if (content ~ /CustomResourceDefinition/) {print content; print "---"}}' > helm/crds/crds.yaml
```

# Good for first issue?

* cluster-wide resources installation fails when installed multiple times from namespace-scoped helm installation
`helm install` prévoit une option -n pour déclarer un namespace ainsi que `--create-namespace` ce qui est top sauf que certains Charts installent des ressources "cluster-wide" e.g. CustomResourceDefinition, ClusterRoles. 
Ainsi, lorsque l'on essaie de déployer 2 fois le meme chart, helm nous retourne une erreur `Error: Unable to continue with install: CustomResourceDefinition "backups.postgresql.cnpg.io" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: key "meta.helm.sh/release-namespace" must equal "doubbz-client-2": current value is "doubbz-client-1"`
Idéalement, les ressources cluster-wide devraient etre non bloquantes. Ceci dit, est ce que cela n'entrainerait des regressions sur la plupart des charts helm open source ?

* pas assez de hook pour gérer l'install de dépendances
Dans l'ordre, en general on veut install les charts dependencies (i.e. clusters operators e.g. cnpg, rabbitmq) -> puis les operands (e.g. la DB PG, le cluster rabbitmq) ce qui va également générer des secrets avec les credentials pour communiquer avec ces composants -> et enfin les applications qui en dépendent
Idée : il aurait fallu essayer de déployer les DB en post-install post-upgrade peut etre... est ce que helm n'aurait pas pu se demmerder tout seul avec les secrets ?

* il faudrait une equivalence pour l'option --from-env-file dans helm. Bien résumé dans cette issue cf. https://github.com/helm/helm/issues/7772

# Keep in mind

* Just like in go, variables are pointers
I've discovered it pretty late: setting values inside templates can mutate source dictionnaries!!! Fortunately i didn't break anything because i was careful but it seems pretty easy to ship a major bug this way.

For example, second line here
```yaml
{{- $client := .Values.client -}}
{{- $_ := unset $client "name" -}}
```
will overwrite name value for every template (in this case it will "remove" the value).
We can use many functions such as deepCopy, mustDeepCopy, Omit...
