---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Como usar el módulo 'stat' de Ansible dentro de un Loop"
subtitle: "La mejor manera de usar el módulo stat dentro de un loop"
summary: "La mejor manera de usar el módulo stat dentro de un loop"
authors: [luiscachog]
tags: [Ansible, Jinja2]
categories: [Ansible, DevOps]
date: 2020-04-12
lastmod: 2020-04-12
publishDate: 2020-04-12
featured: true
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

{{% toc %}}

# Como usar el módulo "stat" de Ansible dentro de un Loop

Hola de nuevo, ha pasado mucho tiempo desde la última vez que escribí.
Esta ves estaba trabajando en un playbook de Ansible y me enfrente a lo siguiente:

## El Problema

Dentro de mi playbook de ansible yo quiero verificar si un archivo existe. Dependiendo de la esa salida que la registro en una variable quiero realizar mas acciones de manera condicional.
Para mi caso en especifico, necesito ocupar el modulo `stat` en conjunto con el modulo `file` que es el que define el estado de mi siguiente tarea.

Un gráfico de lo que quiero lograr es:

{{< diagram >}}
graph TD;
A(Stat sobre el archivo) --> B{Existe el archivo?};
B -->|Si| C[state: file];
B -->|No| D[state: touch];
{{< /diagram >}}

## La Solución

A continuación pongo la solución que funciono para mi usando `loops`, `loop_control`, y filtros del tipo `jinja`:

```yaml
- name: Stat over the files
  stat:
    path: "{{ my_loop }}"
    loop:
      - /etc/cron.allow
      - /etc/at.allow
    loop_control:
      loop_var: my_loop
    register: my_stat_var

- name: Create a file if not exists
  file:
    path: "{{ item.my_loop }}"
    owner: root
    group: root
    mode: og-rwx
    state: "{{ 'file' if item.stat.exists else 'touch' }}"
  loop: "{{ my_stat_var.results }}"
```

Básicamente estoy usando la nueva [sintaxis](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html) del `loop` para iterar sobre rodos los archivos que necesito validar.
Con la nueva sintaxis es necesario ocupar las nuevas variables `loop_control` y `loop_var` para evitar advertencias usando el antigui `item`. Ambas variables `loop_control` y `loop_var` controlan el comportamiento del loop y para este caso especificio, en lugar de ocupar la variable `item` voy a sustituirla con alguna que yo defina en `loop_var` para este ejemplo es `my_loop` (Recuerden esto, por que lo vamos a usar en la nueva tarea)

Dentro de la tarea, voy registrando la salida del `stat` en una variable, para este ejemplo `my_stat_var`.

Aqui está el ejemplo de la salida de `my_stat_var`, cuando los dos archivos a verificar **no existen**

```json
ok: [instance-amazon2] => {
    "my_loop": {
        "changed": false,
        "msg": "All items completed",
        "results": [
            {
                "ansible_loop_var": "my_loop",
                "at_cron_restricted_touch": "/etc/cron.allow",
                "changed": false,
                "failed": false,
                "invocation": {
                    "module_args": {
                        "checksum_algorithm": "sha1",
                        "follow": false,
                        "get_attributes": true,
                        "get_checksum": true,
                        "get_md5": false,
                        "get_mime": true,
                        "path": "/etc/cron.allow"
                    }
                },
                "stat": {
                    "exists": false
                }
            },
            {
                "ansible_loop_var": "my_loop",
                "at_cron_restricted_touch": "/etc/at.allow",
                "changed": false,
                "failed": false,
                "invocation": {
                    "module_args": {
                        "checksum_algorithm": "sha1",
                        "follow": false,
                        "get_attributes": true,
                        "get_checksum": true,
                        "get_md5": false,
                        "get_mime": true,
                        "path": "/etc/at.allow"
                    }
                },
                "stat": {
                    "exists": false
                }
            }
        ]
    }
}
```

En la siguiente tarea "Create a file if not exists" accesso a los resultados registrados en la variable `my_stat_var` de acuerdo a la salida anterior, voy a necesitar usar la variable `results` para accesarla y eso me dará 2 arreglos, uno por cada archivo que estoy verificando.

Además, extraigo la variable `path` de la misma variable `my_stat_var.results`, **PERO**,y está es la razón por la cual escribo este post, dicha variable es parte del loop, entonces necesito accessarla usando `item.my_loop` ya que en la segunda tarea no estoy usando la variable `loop_control`, y de la misma forma puedo accesar usando `item.item` pero prefiero usar la primera sintaxis. En caso de que implementen la variable `loop_control` deberian de poder accesar a los datos usando `my_second_loop.my_loop`.

El siguiente paso es extraer si el archivo existe o no con la variable `item.stat.exists` **PERO**, de nuevo aqui esta el truco, accesamos a ella usando un filtro de `jinja2`, de esa manera podemos evaluar y seleccionar la opción correcta para el `state`

```yaml
# When the file exists
state: file
# When the file does not exists
state: touch
```

Usando esta estrategía brindamos idempotencia a nuestras tareas, por que si el archivo no existe en la primera iteracción va a crear el archivo, de manera que la siguiente vez que corras tu tarea de ansible como el archivo ya existe solamente realiza un `stat` y va a regresar el path del archivo.

Eso es todo por ahora, nos vemos pronto!
