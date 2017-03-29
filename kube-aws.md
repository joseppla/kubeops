#### Kubernetes en AWS

La primera vez que leí una documentación explicando la manera de deployar un cluster de Kubernetes se me quitaron las ganas de probarlo de inmediato. 

A través del manual debian realizarse innumerables instalaciones, configuraciones a mano, tener que montar un cluster de ETCD y un largo etcétera
de tareas que realizar antes de ponder empezar a jugar con ese gestor de contenedores.

<!-- TEASER_END -->

###### Navegando a la deriva...

Al cabo de pocos dias, me volví a poner manos a la obra, posiblemente alguien hubiera hecho una herramienta que me facilitara la faena de deployar un cluster hecho en Google en Amazon que es donde muchos de nosotros trabajamos dia a dia. Afortunadamente la comunidad que se dedica a desarrollar para Kubernetes es extensa y activa con lo que no tardé en encontrar varias soluciones. Nombraré algunas de ellas pero no entraré a fondo en ninguna excepto 'kube-aws' de CoreOS... las otras como KOPS, kubeup.sh  y alguna mas me las reservo para futuros posts.

###### kube-aws o pescando atunes en el paraíso 

Una de las herramientas que encontré es kube-aws de CoreOS, una herramienta pensada para deployar un cluster de Kubernetes en Amazon basandose en Cloudformation.

Como sobre Cloudformation ya sabia bastante como para que no me diera miedo hurgar en el inmenso template en json que escupia esta herramienta y sabiendo que también se podia lanzar sin pasar por cloudformation me puse a mirarla a fondo, al poco rato tenia creado un template de cloudformation el cual podía modificar a mi gusto, instalarlo en una VPC ya creada, cambiar las instancias, no entraré en mas detalle ya que sería un post de Cloudformation y no es el caso.

Digamos que a los diez minutos de trastear con la herramienta tenia un cluster de Kubernetes en HA con 2 masters, 6 workers replartidos en zonas distintas y un ETCD de 3 nodos, mas que suficiente para empezar a jugar.

###### Faenando en alta mar

Una vez realizadas las comprobaciones de que con el cliente en linia podia acceder al cluster me puse a mirar como desplegar una aplicación sencilla, un clásico, wordpress con mysql.

Aquí la cosa ya costó un poco, la documentación de Kubernetes no es que sea muy clara en algunos aspectos, lo básico esta cubierto, lo compliado a veces no explicado con detalle o completamente y lo último que se ha añadido a veces no esta en la documentación oficial y hay que ir buscando por github y los foros donde sin muchas complicaciones se encuentra lo que se necesita. Tengo que decir que mis relaciones con YAML son un poco desastrosas, lo siento tengo que reconozer que me cuesta y mas viniendo del json, la costumbre suele jugar malas pasadas. 


###### Divisando tierra firme

Después de pelearme un poco con el YAML y queriendo deployar usando lo último que habia implementado Kubernetes conseguí lanzar un Wordpress conectado a un MySql y dicho blog publicado a través de un ELB de Amazon, solo me faltaba apuntar un DNS. Pero eso no es ni de lejos lo mas interesante de Kubernetes. Tienes autodiscovery de salida, rolling updates configurables, healthchecks, logs centralizados, monitorización y en las últimas versiones puedes tener Contenedores con estado "StatefulSets", conectar volumenes EBS automaticamente y muchas mas funcionalidades. 



###### Entrando en detalle 

A partir de aquí voy a explicar paso a paso con detalle todo el proceso de instalación, par los que sean vagos ya aviso que el artículo va a ser bastante extenso, ahora bien, una vez finalizado tendremos a nuestra disposición un cluster de Kubernetes preparado para deployar servicios en producción usando ELBs de Amazon, con un sistema de autoregistro de los servicios en Route53 y alguna cosa mas, en el tintero se quedaran el autoscaling de los nodos, auto-recuperación del ETCD y alguna cosa mas que ya cubriremos en futuros posts o que se pueden obtener con una pequeña búsqueda por internet.


###### Preparando el barco para zarpar

Lo primero que tenemos que hacer para poder empezar a definir como será nuestro cluster es bajarnos la herramienta de github, la podemos encontrar en https://github.com/coreos/kube-aws. La podemos poner en una carpeta de nuestra elección o copiarla en /usr/bin para mayor sencillez, la marcamos como ejecutable con el clásico chmod +x y ya estamos listos para empezar.


###### Inicialización del template
El primer paso para empezar a definir nuestro cluster es inicializar un fichero de template que se usara para genenrar el fichero de Cloudormation final, para ello debemos ejecutar el siguiente comando:

kube-aws init

Al ejecutarlo si no ponemos ningún parámetro nos dará un error y nos pedirá algunos parámetros necesarios para su correcto funcionamiento que seran mas o menos estos:

"--cluster-name", "--external-dns-name", "--region", "--availability-zone"

El nombre del cluster, el nombre externo dns (importante que resuleva pq es el que usaran los nodos para la conexión con el master), la región de Amazon y la zona de disponibilidad deseada.

¿Porqué la zona de disponibilidad si queremos montar un cluster en HA? os preguntareis.... pues eso también me pregunto yo pero se que en las primeras versiones de esta herramienta no era posible montar un cluster en HA y creo que se han dejado este parámetro como obligatorio cuando no debería serlo como veremos mas adelante.

Si ponemos los parametros que nos faltan obtendremos esto:
```
kube-aws init --cluster-name k8s-edyo --external-dns-name k8s-edyo.kubeops.net --region us-east-1 --availability-zone us-east-1a
Success! Created cluster.yaml

Next steps:
1. (Optional) Edit cluster.yaml to parameterize the cluster.
2. Use the "kube-aws render" command to render the CloudFormation stack template and coreos-cloudinit userdata.
```

Esto nos creará un fichero tipo template que ahora modificaremos par adaptar a nuestras necesidades, se puede ya lanzar tal y como esta pero siempre es bueno pegarle un vistazo para ajustar tipo de maquinas que compondran el cluster y algun parametro sobre las subnets para tener alta disponibilidad.

Como el tamaño del archivo es importante, voy a quitar todos los comentarios y opciones que no tocaremos en este artículo, de todos modos es interesante leer el fichero original ya que hay muchisimas opciones que igual no tocamos hoy pero son de gran interes para un deploy en producción. Voy comentando dentro del fichero para simplificar el seguimiento de los parámetros.


```
# El nombre del cluster que le hemos pasado por parámetro.
clusterName: k8s-edyo

# El nombre fqdn que le hemos pasado por parámetro.
externalDNSName: k8s-edyo.kubeops.net

# Descomentamos este, la rama de la release de la versión de CoreOS 
#(el SO que usaremos para deployar los nodos).
releaseChannel: stable

# Descomentamos y ponemos a true para que la herramienta nos cree la entrada DNS en Route53.
createRecordSet: true

# Descomentamos y ponemos el ZoneID de route53 del dominio anterior.
hostedZoneId: "AIX95O34ZN4A"

# Descomentamos y ponemos el nombre de una llave SSH existente en AWS.
keyName: 'edyo-admin'

# La zona que hemos pasado por paràmetro
region: us-east-1

# La AZ que hemos pasado por paràmetro, la comentamos para obtener HA.
#availabilityZone: us-east-1a

# El ARN de la key KMS de AWS, es una clave que podemos generar en el panel de IAM de AWS y nos servirá 
# para encriptar todas las comunicaciones entre masters, nodos y etcd, descomentamos y ponemos un ARN 
# existente o generamos uno nuevo.
kmsKeyArn: ""

# Descomentamos este, el número de masters que queremos, como vamos a deployar solo en 2 
# zonas, lo pondremos a 2 aunqué podríamos poner tranquilamente 4
controllerCount: 2

# Descomentamos este, es el tipo de instancia que vamos a usar para los masters, no pongais nada mas pequeño
# que el default o no arrancara el cluster, para nuestro propósito pondremos m3.medium.
controllerInstanceType: m3.medium

# Descomentamos esta parte de los worker pools para que nos quede así,un pool llamado nodepool1 y ponemos el count a cuatro nodos que se crearan repartidos entre las 2 subnets.

worker:
  nodePools:
  - name: nodepool1
    count: 4

# Descomentamos este, el tipo de instancia de los workers, según lo que queramos deployar ha de ser mayor o 
# menor, para nuestro proposito pondremos las mismas que en los masters pero según lo que queramos han de ser
# de mayor o menor tamaño (para cosas mas serias mirar los pools de maquinas donde podremos especificar 
# varios tipos).
workerInstanceType: m3.medium

# Descomentamos este si quieremos que los workers tengan un tamaño de disco específico, no es aconsejable 
# guardar datos en los discos de los nodos, mejor los subimos a 50G pero no los usaremos.
workerRootVolumeSize: 50

# Descomentamos este, parámetro importante, el número de nodos en el cluster etcd, aquí subiremos de 1 a 3 
# para tener alta disponibilidad, hay que tener en cuenta que kubernetes depende completamente de este 
# cluster y a ser posible debería montarse por separado y tener algun sistema de auto-recovery.
etcdCount: 3

# Descomentamos este, cambiaremos el tipo de instancia de los nodos de etcd a m3.medium para un mayor 
# rendimiento aunque realmente para esta demo no lo necesitemos.
etcdInstanceType: m3.medium

# Descomentamos este si queremos especificar el CIDR del VPC que se creará, par deployar en un VPC existente
# leer el fichero original.
vpcCIDR: "10.0.0.0/16"

# Descomentamos estos, aquí se pueden describir cuantas subnets queremos y de que tipo, públicas o privadas,
# para esta demo usaremos dos públicas però las best practices de AWS no aconsejan este tipo de deployment.
subnets:
  - name: ManagedPublicSubnet1
    private: false
    availabilityZone: us-east-1c
    instanceCIDR: "10.0.10.0/24"
  - name: ManagedPrivateSubnet1
    private: false
    availabilityZone: us-east-1e
    instanceCIDR: "10.0.20.0/24"

# Descomentamos este, es el tiempo que durará el certificado de los nodos , etcd, masters, mejor curarnos en
# salud y ponerlo igual que la CA a 10 años
tlsCertDurationDays: 3650

# Descomentamos este si queremos cambiar la versión del SO base, actualizaremos a la última versión estable.
kubernetesVersion: v1.5.3_coreos.0

# Descomentamos estos para poner tags al cluster y asi podelos diferenciar cuanto tengamos varios clusters, 
# realmente no es necesario si deployamos uno pero así vemos como queda todo
stackTags:
  Name: "Kubernetes"
  Environment: "Edyo"
```

Estos son los parámetros básicos que creo importantes para una primera toma de contacto de Kubernetes en AWS, hay bastantes mas que cubriremos en pròximos artículos, esta herramienta dispone de demasiadas opciones para tocarlas todas de golpe y además explicar algo de Kubernetes en un solo post, sinceramente creo que será demasiado largo y mas del 80% de la gente optará por parar ahora mismo, no lo hagais, vale la pena :)

Dicho esto repasemos rapidamente lo que tenemos, vamos a crear un cluster con máquinas m3.medium para todos los nodos (masters, etcd y workers), con DNS, que seran deployados en 2 subnets públicas en un VPC nuevo y creando un registro DNS en un dominio que ya tenemos apuntado a Route53, ojo con el dominio y la key KMS. Sin estos recursos el cluster no va a funcionar y os frustrareis antes de tiempo ;-)

###### Arrancando motores
Una vez tenemos el template modificado a nuestro gusto pasamos a la generación de lo que realmente será el fichero de configuración que kube-aws usará para lanzar el cluster en Amazon, dicho archivo no es un Cloudformation pero deduzco que por detras creará uno que si que lo es a partir de este y lo lanzará, deduzco porqué como podemos optar por exportar todo a un Cloudformation en vez de lanzar el cluster desde la herramienta sospecho que es lo que hace el programa por detrás.

Para generar el "template" final y varios archivos extras que necesita la aplicación ejecutaremos el siguiente comando aunque nos dice que será deprecado en breve:

```
kube-aws render
```
 
Esto nos va a generar una estructura de directorios como esta:
```
.
+----cluster.yaml
+----credentials
| +---ca-key.pem
| +----apiserver.pem
| +----worker-key.pem
| +----admin.pem
| +----etcd-key.pem
| +----etcd-client.pem
| +----etcd-client-key.pem
| +----ca.pem
| +----.gitignore
| +----apiserver-key.pem
| +----worker.pem
| +----admin-key.pem
| +----etcd.pem
+----kubeconfig
+----userdata
| +----cloud-config-controller
| +----cloud-config-etcd
| +----cloud-config-worker
+----stack-templates
| +----node-pool.json.tmpl
| +----control-plane.json.tmpl
| +----root.json.tmpl
```

Veremos que se han creado 4 directorios nuevos:

* Credentials: contiene las keys que se usaran para la comunicacion segura entre todas las partes del cluster.
* kubeconfig: contiene la configuración para el cliente de Kubernetes.
* userdata: contiene todos los userdata para los masters, workers,etc.
* stack-templates: contiene todos los templates para generar los cloudformations necesarios.```
kube-aws up --s3-uri s3://kube-aws-edyo

Con esto ya tenemos todos los pasos para deployar un cluster de kubernetes en versión development o test en AWS.

Para poder arrancar el cluster necesitamos un bucket de S3 donde se van a subir todas las definiciones de Cloudformation necesarias, en estas últimas versiones kube-aws crea un stack que lleva dentro 2 o más stacks, uno para lo que le llaman el 'Control Plane' compuesto por los masters y los ETCDs y otro que sera el nodepool que hemos creado, si tuvieramos mas nodepools se crearía un stack por cada uno de ellos para facilitar su posterior modificación sin afectar al resto del cluster.

Dicho esto ya solo nos queda lanzar el comando de creación del stack apuntando a un bucket de S3 ya existente:
```
kube-aws up --s3-uri s3://kube-aws-edyo
```
A partir de aquí podremos seguir la creación de todos estos stacks desde el dashboard de AWS Cloudformation, para los que no esteis familiarizados con esta herramienta solo hay que marcar cada uno de los stacks y en la pestaña de events podremos ir viendo los recursos que se van creando y su estado. Si todo v abién en unos 10 minutos deberíamos tener listo nuestro cluster de Kubernetes.


###### kubectl,  la herramienta de interacción con el cluster




