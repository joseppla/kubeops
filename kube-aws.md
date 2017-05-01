## Kubernetes en AWS

Hace un año atrás desplegar un cluster de Kubernetes desde cero no era tarea fácil, tenias que hacerlo a mano, crear tu propia herramienta o usar alguna solución ya existente que no estaba ni completa ni muy bien documentada. Kubernetes era algo reservado solo a unos pocos expertos o apasionados de las nuevas tecnologías que se aventuraban a probar algo no del todo 'terminado'. Si lo que querías era un cluster en HA, resistente a fallos, con auto escalado, en tu hosting habitual y con el nivel de seguridad/fiabilidad de un entorno de producción la cosa se volvía imposible. Mucha gente, entre los cuales me incluyo, desistía al ver la faena que se le venía encima sin ver las ventajas que consigo traía. No había posibilidad de tener multizona en Amazon, aún menos multimaster y otras tantas cosas que hacían que uno no se animara a probar.

<!-- TEASER_END -->

Por esas fechas empecé a trabajar en un proyecto basado en contenedores y revisé algunas de las herramientas existentes que desplegaban Kubernetes en Amazon, al no convencerme ninguna de ellas me aventuré a desplegar los servicios usando Amazon ECS(la solución de Amazon para contenedores). Sinceramente ECS tampoco estaba muy avanzado en esos tiempos pero me permitió olvidarme de masters y poder tener algo corriendo en multizona. No tardé mucho en descubrir que ECS tampoco era la panacea, tenía otros muchos problemas que iban saliendo a medida que lo usabas y crecía la complejidad del despliegue o sus necesidades...seguía con esa sensación de decepción ¿Cómo podía ser posible que nadie hubiera creado alguna herramienta para facilitar un despliegue decente de Kubernetes en Amazon?

Pasado un tiempo de mi primer intento y sintiéndome decepcionado por ECS volví a revisar las herramientas que anteriormente había visto, me centré en 2 de ellas, Kops y kube-aws. Kops desplegaba un cluster de Kubernetes en Amazon usando Terraform (infrastructure as code de Hashicorp) y kube-aws hacía más o menos lo mismo pero usando Cloudformation(infrastructure as code de Amazon). Las dos tenían aún mucho camino por recorrer, no disponían de multimaster, tampoco tenían un deploy decente de ETCD(parte esencial de Kubernetes) y otros tantos problemas pero ya pintaban un poco mejor que hacía unos meses. Después de probarlas a fondo y valorar que me convencía más de cada una me decanté por kube-aws. Un servidor ya tenia Cloudformation por la mano y la infraestructura de mi empresa también lo usaba con lo que lo más lógico era usar kube-aws. En aquel momento ya permitía desplegar nodos en multizona, aún le faltaba multimaster y muchas cosas pero la velocidad a la que se le añadían features y el roadmap de futuras versiones me hicieron ver que, si bien algunas de mis necesidades no estaban cubiertas en ese momento, lo estarían en poco tiempo. También contaba con la ventaja de que al basarse en Cloudformation una vez realizado el template podía modificar el fichero para ajustar, añadir o quitar lo que me interesara.

#### kube-aws o pescando atunes en el paraíso

Una vez decidida la herramienta me puse manos a la obra y a los 15 minutos de trastear con kube-aws disponía de un cluster de Kubernetes corriendo en Amazon con un master y 6 nodos repartidos entre 2 zonas.

Una vez realizadas las comprobaciones conexión al cluster desde el cliente de Kubernetes me puse a mirar como desplegar una aplicación sencilla, un clásico, Wordpress con Mysql. Aquí la cosa ya costó un poco, la documentación de Kubernetes no era muy clara en algunos aspectos, lo básico estaba cubierto, lo complejo a veces no explicado con detalle o incompleto y las últimas features añadidas ni aparecían. A veces había que ir buscando por Github y los foros, donde sin muchas complicaciones se acababa encontrando lo que se necesitaba. Después de pelearme un poco con los ficheros de configuración en YAML y queriendo deployar usando las últimas features añadidas conseguí lanzar un Wordpress conectado a un MySql y dicho servicio publicado a través de un ELB de Amazon.

Afortunadamente las cosas han cambiado y las herramientas han madurado lo suficiente como para que los creadores de Kubernetes decidan substituir su propia herramienta "kubeup.sh" en favor de estas 2 anteriormente mencionadas. Las dos disponen de multimaster, multizona y bastantes más funcionalidades interesantes.

#### La potencia de Kubernetes

Sinceramente lo que desplegué en ese momento y que me pareció una genialidad hoy en día me parece de primaria. Hay tantas cosas que están tan bien pensadas en Kubernetes y que facilitan tanto las tareas que vale la pena dedicarle tiempo a entender que es, como funciona, que partes lo componen y las ventajas que consigo trae. Entre algunas de ellas y para no extenderme diré que tiene de salida, dejándome muchas en el tintero, estas features:

- Autodiscovery
- Montaje de volúmenes (Amazon EBS/EFS, GCE volumes)
- Ficheros de configuración actualizables en caliente
- Rolling updates y rollbacks
- Healthchecks
- Autoscaling
- Gestión de secrets
- Tareas programadas
- Publicación de servicios con AWS ELB o GCE Ingress
- Monitoring
- Logging centralizado
- Balanceo de carga
- HA de servicios
- Mucho más...


### Entrando en detalle 

Ahora nos pondremos serios, a partir de aquí voy a explicar paso a paso con detalle todo el proceso de instalación, para los que sean vagos ya aviso que el artículo va a ser bastante extenso ahora bien, una vez finalizado tendremos a nuestra disposición un cluster de Kubernetes en HA preparado para deployar servicios en un entorno de pruebas/staging usando Amazon. En el tintero se quedaran el autoscaling de los nodos, auto-recuperación del ETCD, registro automático de DNS para los servicios y muchas cosas más que ya cubriremos en futuros posts o que se pueden aprender con una pequeña búsqueda por Internet y paciencia.


#### Preparando el barco para zarpar

Doy por supuesto una serie de conocimientos en Amazon que no cubriré aquí ya que no es el propósito de este blog ni de este artículo, algunos de ellos se pueden aprender en poco tiempo y otros son más costosos, que nadie desista en intentarlo ya que no todos son necesarios si lo único que queremos hacer es desplegar algo par empezar a jugar con Kubernetes.

También será necesario disponer del cliente de AWS configurado con una access key id y secret access key con los permisos necesarios para que la herramienta pueda realizar cambios en AWS, si no es así el primer paso es crear un usuario nuevo y configurar el cliente de AWS en línea de comandos, toda la información necesaria se puede encontrar en la web de Amazon. También necesitaremos una clave KMS que podemos crear desde la consola de IAM de AWS así como una clave SSH para poder acceder a los nodos y un bucket de S3 para subir ficheros.

Lo primero que tenemos que hacer para poder empezar a definir como será nuestro cluster es bajarnos la herramienta kube-aws de Github que se puede  encontrar en https://github.com/coreos/kube-aws. La podemos poner en una carpeta de nuestra elección o copiarla en /usr/bin para mayor facilidad, la marcamos como ejecutable con el clásico chmod +x y ya estamos listos para empezar.


#### Inicialización del template y personalización

El primer paso para empezar a definir nuestro cluster en HA es inicializar un fichero de template que se usara para generar el fichero de configuración final, para ello creamos un directorio vacío y ejecutamos el siguiente comando:
```
kube-aws init
```
Al ejecutarlo si no ponemos ningún parámetro nos dará un error y nos dirá que hay algunos parámetros necesarios para su correcto funcionamiento que serán más o menos estos:
```
"--cluster-name", "--external-dns-name", "--region", "--availability-zone"
```
El nombre del cluster, el nombre externo DNS (importante que resuelva ya que es el que usaran los nodos para la conexión con el master), la región de Amazon y la zona de disponibilidad deseada. Hago hincapié en que en el external DNS name deberíamos poner un nombre de un dominio existente y si puede ser que este en Route53, de esta manera podemos decirle a la herramienta que nos cree automáticamente el registro en Route53 y no tendremos que crearlo nosotros a posteriormente.

¿Porqué la zona de disponibilidad si queremos montar un cluster en HA? os preguntareis.... pues eso también me pregunto yo, sé que en las primeras versiones de esta herramienta no era posible montar un cluster en HA y creo que se han dejado este parámetro como obligatorio cuando no debería serlo como veremos más adelante.

Si ponemos los parámetros que nos faltan obtendremos esto:
```
kube-aws init --cluster-name k8s-kubeops --external-dns-name k8s.kubeops.net --region us-east-1 --availability-zone us-east-1a
Success! Created cluster.yaml

Next steps:
1. (Optional) Edit cluster.yaml to parameterize the cluster.
2. Use the "kube-aws render" command to render the CloudFormation stack template and coreos-cloudinit userdata.
```

Esto nos creará un fichero tipo template que ahora modificaremos par adaptar a nuestras necesidades, se puede ya lanzar tal y como está pero siempre es bueno pegarle un vistazo para ajustar tipo de maquinas que compondrán el cluster y algún parámetro sobre las subnets para tener alta disponibilidad.

Como el tamaño del archivo es importante, voy a quitar todos los comentarios y opciones que no tocaremos en este artículo, de todos modos es interesante leer el fichero original ya que hay muchísimas opciones que igual no tocamos hoy pero son de gran interés para un deploy en producción. Voy comentando dentro del fichero para simplificar el seguimiento de los parámetros.


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

# La zona que hemos pasado por parámetro
region: us-east-1

# La AZ que hemos pasado por parámetro, la comentamos para obtener HA.
#availabilityZone: us-east-1a

# El ARN de la key KMS de AWS, es una clave que podemos generar en el panel de IAM de AWS y nos servirá 
# para encriptar todas las comunicaciones entre masters, nodos y etcd, descomentamos y ponemos un ARN 
# existente o generamos uno nuevo.
kmsKeyArn: ""

# Descomentamos este, el número de masters que queremos, como vamos a deployar solo en 2 
# zonas, lo pondremos a 2 pero podríamos poner tranquilamente 4
controllerCount: 2

# Descomentamos este, es el tipo de instancia que vamos a usar para los masters, no pongáis nada mas pequeño
# que el default o no arrancara el cluster, para nuestro propósito pondremos m3.medium.
controllerInstanceType: m3.medium

# Descomentamos esta parte de los worker pools para que nos quede así,un pool llamado nodepool1 y ponemos el count a cuatro nodos que se crearan repartidos entre las 2 subnets.

worker:
  nodePools:
  - name: nodepool1
    count: 4

# Descomentamos este, el tipo de instancia de los workers, según lo que queramos deployar ha de ser mayor o 
# menor, para nuestro propósito pondremos las mismas que en los masters pero según lo que queramos han de ser
# de mayor o menor tamaño (para cosas más serias mirar los pools de maquinas donde podremos especificar 
# varios tipos).
workerInstanceType: m3.medium

# Descomentamos este si queremos que los workers tengan un tamaño de disco específico, no es aconsejable 
# guardar datos en los discos de los nodos, mejor los subimos a 50G pero no los usaremos.
workerRootVolumeSize: 50

# Descomentamos este, parámetro importante, el número de nodos en el cluster etcd, aquí subiremos de 1 a 3 
# para tener alta disponibilidad, hay que tener en cuenta que Kubernetes depende completamente de este 
# cluster y a ser posible debería montarse por separado y tener algún sistema de auto-recovery.
# Update: en las últimas versiones beta ya está montado por separado y lleva ASG y auto-recovery.
etcdCount: 3

# Descomentamos este, cambiaremos el tipo de instancia de los nodos de etcd a m3.medium para un mayor 
# rendimiento aunque realmente para esta demo no lo necesitemos.
etcdInstanceType: m3.medium

# Descomentamos este si queremos especificar el CIDR del VPC que se creará o para deployar en un VPC existente
# leer el fichero original.
vpcCIDR: "10.0.0.0/16"

# Descomentamos estos, aquí se pueden describir cuantas subnets queremos y de que tipo, públicas o privadas,
# para esta demo usaremos dos públicas pero las best practices de AWS no aconsejan este tipo de deployment.
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

# Descomentamos estos para poner tags al cluster y asi poderlos diferenciar cuanto tengamos varios clusters, 
# realmente no es necesario si deployamos uno pero así vemos como queda todo
stackTags:
  Name: "Kubernetes"
  Environment: "Kubeops"
```

Estos son los parámetros básicos que creo importantes para una primera toma de contacto de Kubernetes en AWS, hay bastantes más que cubriremos en próximos artículos, esta herramienta dispone de demasiadas opciones como para aprenderlas todas de golpe y además explicar algo de Kubernetes en un solo post, sinceramente creo que será demasiado largo y más del 80% de la gente optará por parar ahora mismo, no lo hagáis, vale la pena :)

Dicho esto repasemos rápidamente lo que tenemos, vamos a crear un cluster con máquinas m3.medium para todos los nodos (masters, etcd y workers), con DNS, que serán deployados en 2 subnets públicas en un VPC nuevo y creando un registro DNS en un dominio que ya tenemos apuntado a Route53, ojo con el dominio y la key KMS. Sin estos recursos el cluster no va a funcionar y os frustrareis antes de tiempo ;-)

#### Arrancando motores
Una vez tenemos el template modificado a nuestro gusto pasamos a la generación de lo que realmente será el fichero de configuración que kube-aws usará para lanzar el cluster en Amazon, dicho archivo no es un Cloudformation pero deduzco que por detrás creará uno que si que lo es a partir de este y lo lanzará, lo deduzco porqué podemos optar por exportar todo a un Cloudformation en vez de lanzar el cluster desde la herramienta y sospecho que es lo que hace el programa por detrás.

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

* Credentials: contiene las keys que se usaran para la comunicación segura entre todas las partes del cluster.
* kubeconfig: contiene la configuración para el cliente de Kubernetes.
* userdata: contiene todos los userdata para los masters, workers,etc.
* stack-templates: contiene todos los templates para generar los Cloudformations necesarios.```

Con esto ya tenemos todos los pasos para deployar un cluster de Kubernetes en versión development o test en AWS.

Para poder arrancar el cluster necesitamos un bucket de S3 donde se van a subir todas las definiciones de Cloudformation necesarias, en estas últimas versiones kube-aws crea un stack que lleva dentro 2 o más stacks, uno para lo que le llaman el 'Control Plane' compuesto por los masters y los ETCDs y otro que sera el nodepool que hemos creado, si tuviéramos más nodepools se crearía un stack por cada uno de ellos para facilitar su posterior modificación sin afectar al resto del cluster.

Dicho esto ya solo nos queda lanzar el comando de creación del stack apuntando a un bucket de S3 ya existente:
```
kube-aws up --s3-uri s3://kube-aws-kubeops
```
A partir de aquí podremos seguir la creación de todos estos stacks desde el dashboard de AWS Cloudformation, para los que no estáis familiarizados con esta herramienta solo hay que marcar cada uno de los stacks y seleccionando la pestaña de events podremos ir viendo los recursos que se van creando y su estado. Si todo va bien en unos 10 minutos deberíamos tener listo nuestro cluster de Kubernetes. En su defecto esperaremos a que la herramienta nos diga que el cluster ya está creado y nos lo creeremos ;-)


#### kubectl,  la herramienta de interacción con el cluster

Finalmente y pare poder verificar que todo ha ido correcto y de que disponemos de nuestro cluster vamos a descargarnos la herramienta de interacción desde la línea de comandos, podríamos hacer uso del UI Web desde el cual se pueden realizar infinidad de cosas pero como siempre la línea de comandos supera a la UI. Nos la podemos descargar con este sencillo comando:
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```
La movemos en nuestro /usr/bin o directorio deseado y la marcamos como ejecutable con el chmod +x.
Una vez tenemos la herramienta debemos tenemos 2 opciones, cada vez que la ejecutamos especificar el fichero de configuración que anteriormente kube-aws nos ha generado(kubeconfig) o hacerlo bien y copiar este fichero y la carpeta de credenciales a una carpeta en nuestro home llamada '.kube' renombrando el fichero kubeconfig a config, lo podemos hacer fácilmente con estos 3 comandos:

```
mkdir ~/.kube
cp kubeconfig ~/.kube/config
cp -r credentials ~/.kube/.
```

Una vez hecho este sencillo paso ya podemos verificar si realmente nuestro cluster está funcionando ejecutando 2 comandos:

```
kubectl get nodes
NAME                          STATUS    AGE
ip-10-0-10-119.ec2.internal   Ready     5m
ip-10-0-10-88.ec2.internal    Ready     6m
ip-10-0-10-91.ec2.internal    Ready     12m
ip-10-0-20-100.ec2.internal   Ready     6m
ip-10-0-20-182.ec2.internal   Ready     6m
ip-10-0-20-7.ec2.internal     Ready     13m
```
Aquí lo que hemos hecho es pedirle a Kubernetes que nos muestre todos los nodos que componen el cluster, podemos ver los 6 nodos, 4 workers y 2 masters. Los nodos de etcd no forman parte de Kubernetes , al menos en este tipo de instalación.


```
kubectl get pods --namespace=kube-system
NAME                                                 READY     STATUS    RESTARTS   AGE
heapster-v1.2.0-4088228293-d33tz                     2/2       Running   0          7m
kube-apiserver-ip-10-0-10-91.ec2.internal            1/1       Running   0          13m
kube-apiserver-ip-10-0-20-7.ec2.internal             1/1       Running   0          14m
kube-controller-manager-ip-10-0-10-91.ec2.internal   1/1       Running   0          15m
kube-controller-manager-ip-10-0-20-7.ec2.internal    1/1       Running   0          14m
kube-dns-782804071-40trf                             4/4       Running   0          16m
kube-dns-782804071-l2bk9                             4/4       Running   0          7m
kube-dns-autoscaler-2813114833-pqvqf                 1/1       Running   0          16m
kube-proxy-ip-10-0-10-119.ec2.internal               1/1       Running   0          8m
kube-proxy-ip-10-0-10-88.ec2.internal                1/1       Running   0          8m
kube-proxy-ip-10-0-10-91.ec2.internal                1/1       Running   0          14m
kube-proxy-ip-10-0-20-100.ec2.internal               1/1       Running   0          8m
kube-proxy-ip-10-0-20-182.ec2.internal               1/1       Running   0          8m
kube-proxy-ip-10-0-20-7.ec2.internal                 1/1       Running   0          15m
kube-scheduler-ip-10-0-10-91.ec2.internal            1/1       Running   0          14m
kube-scheduler-ip-10-0-20-7.ec2.internal             1/1       Running   0          14m
kubernetes-dashboard-v1.5.1-nd69h                    1/1       Running   0          16m
```
Aquí listamos todos los Pods (unidades mínimas de servicio) que están corriendo en Kubernetes en el namespace(sistema para aislar zonas en Kubernetes) de kube-system, que es donde corren los servicios core de Kubernetes, si todos están en Running es buena señal.

Llegados a este punto ya tenemos disponibles un cluster de kubernetes preparado para empezar a jugar, desplegarlo no lleva más de 10-15 minutos y tumbarlo otros tantos, si una vez usado queremos eliminarlo solo tenemos que usar este sencillo comando:
```
kube-aws destroy
```
La herramienta lanzará la orden contra Cloudformation donde podremos ir viendo el progreso de borrado, vale la pena asegurarse de que todo va bien y no nos quedan máquinas encendidas, no queremos tener sorpresas a final de mes cuando la factura de AWS nos llegue ;-)
