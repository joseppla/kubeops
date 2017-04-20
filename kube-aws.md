#### Kubernetes en AWS

 Hace un año atrás deployar un cluster de Kubernetes desde cero no era tarea fácil, tenias que hacerlo a mano, crear tu propia herramienta o usar alguna solución ya existente que no estaba ni completa ni muy bien documentada. Kubernetes era algo reservado solo a unos pocos conocedores del sistema o apasionados de las nuevas tecnologías. Si lo que queríamos era un cluster en HA, resistente a fallos, con autoescalado, en tu hosting habitual y con el nivel de seguridad/fiabilidad de un entorno de producción la cosa ya se volvía casi imposible. Mucha gente, entre los cuales me incluyo, desistía al ver la faena que se le venía encima sin ver las ventajas que consigo traía. Un servidor miro varias veces la documentación para hacerlo pero la complejidad de montarlo a mano, la indecisión de que herramienta usar y la falta de HA me hicieron desistir.

<!-- TEASER_END -->


Afortunadamente hoy en dia existen varias herramientas que nos facilitan esta taréa y se puede decir que algunas de ellas han madurado lo suficiente como para que los creadores de kubernetes decidan substituir su proipia herramienta "kubeup.sh" en favor de estas. Entre las mas conocidas y las únicas recomendadas por Kubernetes estan Kops y kube-aws. Kops es capaz de desplegar un cluster de kubernetes en AWS haciendo uso de Terraform (infrastructura en código de Hashicorp). kube-aws es una herramienta creada por CoreOS que hace uso de Cloudformation (infraestructura en código de Amazon) y también deploya en AWS.


###### kube-aws o pescando atunes en el paraíso 

Como  un servidor ya tenia Cloudformation por la mano y la infrastructura de mi empresa estaba en AWS lo más lógico era elegir kube-aws, no desplegaría un cluster en otro hosting ni sabía nada de Terraform. kube-aws me permitía lanzar el cluster desde la propia herramienta o exportar el fichero a cloudformation el cual podia modificar a posteriori a mi gusto, esa fué realmente la clave de elegir kube-aws. A los quince minutos de haberme puesto a trastear con la herramienta tenia montado un cluster con 2 masters, 6 nodos, etcd y sobre mi VPC de Staging, que más se podía pedir!

Una vez realizadas las comprobaciones de que con el cliente podia acceder al cluster me puse a mirar como desplegar una aplicación sencilla, un clásico, wordpress con mysql.

Aquí la cosa ya costó un poco, la documentación de Kubernetes no es que sea muy clara en algunos aspectos, lo básico esta cubierto, lo complejo a veces no explicado con detalle o incompleto y las últimas features añadidas a veces ni aparecen, hay que ir buscando por Github y los foros donde, sin muchas complicaciones, se acaba encuentrando lo que se necesita.

Después de pelearme un poco con los ficheros de configuración en YAML y queriendo deployar usando las ultimas features que habia implementado Kubernetes conseguí lanzar un Wordpress conectado a un MySql y dicho blog publicado a través de un ELB de Amazon, solo me faltaba apuntar un DNS!


###### La poténcia de Kubernetes

Sinceramente el despliegue que realizé es bastante sencillo una vez entendida la filosofía de Kubernetes y su funcionamiento, pero hay tantas cosas que estan tan bien pensadas que vale la pena dedicarle tiempo a entender que es, como fuciona, que partes lo componen y las ventajas que trae consigo. Entre algunas de ellas y para no extenderme diré que tiene de salida y dejandome muchas en el tintero estas features:

- Autodiscovery
- Montaje de volumenes (Amazon EBS/EFS, GCE volumes)
- Ficheros de configuración actualizables en caliente
- Rolling updates y rollbacks automaticos
- Healthchecks
- Autoscaling
- Gestión de secrets
- Tareas programadas
- Publicación de servicios con AWS ELB o GCE Ingress
- Monitoring
- Logging centralizado
- Balanceo de carga
- HA de servicios
y mucho más


###### Entrando en detalle 

A partir de aquí voy a explicar paso a paso con detalle todo el proceso de instalación, para los que sean vagos ya aviso que el artículo va a ser bastante extenso, ahora bien, una vez finalizado tendremos a nuestra disposición un cluster de Kubernetes en HA preparado para deployar servicios en un entorno de pruebas/staging usando ELBs de Amazon y alguna cosa mas, en el tintero se quedaran el autoscaling de los nodos, auto-recuperación del ETCD, registro automatico de DNS para los servicios y muchas cosas mas que ya cubriremos en futuros posts o que se pueden obtener con una pequeña búsqueda por internet.


###### Preparando el barco para zarpar

Doy por supuesto una série de conocimientos en Amazon que no cubriré aquí ya que no es el propósito de este blog ni de este artículo, algunos de ellos se pueden aprender en poco tiempo y otros son mas costosos, que nadie desista en intentarlo ya que no todos son necesarios si lo único que queremos hacer es deplegar algo par empezar a jugar con Kubernetes.

También será necesario disponer del cliente de aws configurado con una access key id y secret access key con los permisos necesarios para que la herramienta pueda realizar cambios en AWS, si no es así el primer paso es crear un usario nuevo y configurar el cliente de AWS de linea de comandos, toda la información necesaria se puede encontrar en la web de amazon. También necesitaremos una clave KMS que podemos crear desde la consola de IAM de AWS así como una clave SSH para poder acceder a los nodos y un bucket de S3 para subir ficheros.

Lo primero que tenemos que hacer para poder empezar a definir como será nuestro cluster es bajarnos la herramienta kube-aws de github, la podemos encontrar en https://github.com/coreos/kube-aws. La podemos poner en una carpeta de nuestra elección o copiarla en /usr/bin para mayor facilidad, la marcamos como ejecutable con el clásico chmod +x y ya estamos listos para empezar.


###### Inicialización del template y personalización

El primer paso para empezar a definir nuestro cluster en HA es inicializar un fichero de template que se usara para generar el fichero de configuración final, para ello creamos un directorio vacío y ejecutamos el siguiente comando:

kube-aws init

Al ejecutarlo si no ponemos ningún parámetro nos dará un error y nos dirá que hay algunos parámetros necesarios para su correcto funcionamiento que seran mas o menos estos:

"--cluster-name", "--external-dns-name", "--region", "--availability-zone"

El nombre del cluster, el nombre externo dns (importante que resuleva pq es el que usaran los nodos para la conexión con el master), la región de Amazon y la zona de disponibilidad deseada. Hago hincapié en que en el external dns name deberíamos poner un nombre de un dominio existente y si puede ser que este en Route53, de esta manera podemos decirle a la herramienta que nos cree automaticamente el registro en Route53 y no tendremos que crearlo nosotros a posteriori.

¿Porqué la zona de disponibilidad si queremos montar un cluster en HA? os preguntareis.... pues eso también me pregunto yo, se que en las primeras versiones de esta herramienta no era posible montar un cluster en HA y creo que se han dejado este parámetro como obligatorio cuando no debería serlo como veremos mas adelante.

Si ponemos los parametros que nos faltan obtendremos esto:
```
kube-aws init --cluster-name k8s-kubeops --external-dns-name k8s.kubeops.net --region us-east-1 --availability-zone us-east-1a
Success! Created cluster.yaml

Next steps:
1. (Optional) Edit cluster.yaml to parameterize the cluster.
2. Use the "kube-aws render" command to render the CloudFormation stack template and coreos-cloudinit userdata.
```

Esto nos creará un fichero tipo template que ahora modificaremos par adaptar a nuestras necesidades, se puede ya lanzar tal y como esta pero siempre es bueno pegarle un vistazo para ajustar tipo de maquinas que compondran el cluster y algun parametro sobre las subnets para tener alta disponibilidad.

Como el tamaño del archivo es importante, voy a quitar todos los comentarios y opciones que no tocaremos en este artículo, de todos modos es interesante leer el fichero original ya que hay muchisimas opciones que igual no tocamos hoy pero son de gran interes para un deploy en producción. Voy comentando dentro del fichero para simplificar el seguimiento de los parámetros.


```
# El nombre del cluster que le hemos pasado por parámetro.
clusterName: k8s

# El nombre fqdn que le hemos pasado por parámetro.
externalDNSName: k8s.kubeops.net

# Descomentamos este, la rama de la release de la versión de CoreOS 
#(el SO que usaremos para deployar los nodos).
releaseChannel: stable

# Descomentamos y ponemos a true para que la herramienta nos cree la entrada DNS en Route53.
createRecordSet: true

# Descomentamos y ponemos el ZoneID de route53 del dominio anterior.
hostedZoneId: "AIX95O34ZN4A"

# Descomentamos y ponemos el nombre de una llave SSH existente en AWS.
keyName: 'kubeops-admin'

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
# Update: en las últimas versiones beta ya esta montado por separado y lleva ASG y auto-recovery.
etcdCount: 3

# Descomentamos este, cambiaremos el tipo de instancia de los nodos de etcd a m3.medium para un mayor 
# rendimiento aunque realmente para esta demo no lo necesitemos.
etcdInstanceType: m3.medium

# Descomentamos este si queremos especificar el CIDR del VPC que se creará o para deployar en un VPC existente
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

# Descomentamos este si queremos cambiar la versión del SO base.
kubernetesVersion: v1.5.3_coreos.0

# Descomentamos estos para poner tags al cluster y asi podelos diferenciar cuanto tengamos varios clusters, 
# realmente no es necesario si deployamos uno pero así vemos como queda todo
stackTags:
  Name: "Kubernetes"
  Environment: "Kubeops"
```

Estos son los parámetros básicos que creo importantes para una primera toma de contacto de Kubernetes en AWS, hay bastantes mas que cubriremos en pròximos artículos, esta herramienta dispone de demasiadas opciones como para aprenderlas todas de golpe y además explicar algo de Kubernetes en un solo post, sinceramente creo que será demasiado largo y mas del 80% de la gente optará por parar ahora mismo, no lo hagais, vale la pena :)

Dicho esto repasemos rapidamente lo que tenemos, vamos a crear un cluster con máquinas m3.medium para todos los nodos (masters, etcd y workers), con DNS, que seran deployados en 2 subnets públicas en un VPC nuevo y creando un registro DNS en un dominio que ya tenemos apuntado a Route53, ojo con el dominio y la key KMS. Sin estos recursos el cluster no va a funcionar y os frustrareis antes de tiempo ;-)

###### Arrancando motores
Una vez tenemos el template modificado a nuestro gusto pasamos a la generación de lo que realmente será el fichero de configuración que kube-aws usará para lanzar el cluster en Amazon, dicho archivo no es un Cloudformation pero deduzco que por detras creará uno que si que lo es a partir de este y lo lanzará, lo deduzco porqué podemos optar por exportar todo a un Cloudformation en vez de lanzar el cluster desde la herramienta y sospecho que es lo que hace el programa por detrás.

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
kube-aws up --s3-uri s3://kube-aws-kubeops
```
A partir de aquí podremos seguir la creación de todos estos stacks desde el dashboard de AWS Cloudformation, para los que no esteis familiarizados con esta herramienta solo hay que marcar cada uno de los stacks y en la pestaña de events podremos ir viendo los recursos que se van creando y su estado. Si todo v abién en unos 10 minutos deberíamos tener listo nuestro cluster de Kubernetes. En su defecto esperaremos a que la herramienta nos diga que el cluster ya esta creado y nos lo creeremos ;-)


###### kubectl,  la herramienta de interacción con el cluster

Para poder interactuar con Kubernetes es necesaria una herramienta de comandos, miento, se puede hacer a través de una UI que ha ido mejorando notablemente a través de las versiones pero para sacar realmente la potencia de kubernetes y como todos sabemos, las herramientas en linea de comandos suelen ser bastante mas potentes, escriptables, etc,etc.


