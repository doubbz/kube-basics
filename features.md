# COOL FEATURES NOT TO BE FORGOTTEN

* [container env varibales dependent from another](https://kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/#define-an-environment-dependent-variable-for-a-container)
```yaml
env:
    - name: VAR1
      value: toto
    - name: VAR2
      value: prefix-$(VAR1)-suffix
```
