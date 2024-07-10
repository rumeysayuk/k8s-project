# k8s-project

### minikube start --nodes 5

komutu ile 5 node sahip bir cluster oluşturuldu.


### kubectl create namespace test
### kubectl create namespace production

komutları ile test ve production isimli namespaces oluşturuldu.

### rbac şu şekilde. junior ve senior isimli gruplar oluşturuldu ve senior kullanıcı için production namespacesi ve cluster üzerinde yetkiler verildi.

Bu işlemi gerçekleştirmek için:

cd rbac

kubectl apply -f . 

komutu RoleBindings ve ClusterRoleBindings işlemlerini gerçekleştirir.

Bunları görüntülemek için:

kubectl get rolebindings.rbac.authorization.k8s.io -n test

kubectl get rolebindings.rbac.authorization.k8s.io -n production

kubectl get clusterrolebindings.rbac.authorization.k8s.io


Dış dünya ile haberleşmek için ingress controller oluşturulacaktır. Bu işlem için nginx seçilmiştir.

Öncelikle terminalde 
minikube addons enable ingress
komutu çalıştırılı. Bu komut nginx'i sisteme kuruyor.

kubectl get service -A 
komutu ile gereklilikleri için oluşturduğo objeler görüntülenmektedir.


Cluster'da seçilen 3 worker node sadece "production" ortamında deploy edildi ve cluster tarafından oluşturulan podlar schedule edildi. Bunun dışındaki podların bu worker node üstünde oluşturulmamasını sağlandı. Bu işlemi sağlamak için:

$ node1=$(kubectl get no -o jsonpath="{.items[1].metadata.name}")

$ node2=$(kubectl get no -o jsonpath="{.items[2].metadata.name}")

$ node3=$(kubectl get no -o jsonpath="{.items[3].metadata.name}")

$ kubectl taint node $node1 tier=production:NoSchedule

$ kubectl taint node $node2 tier=production:NoSchedule

$ kubectl taint node $node3 tier=production:NoSchedule

$ kubectl label node $node1 tier=production

$ kubectl label node $node2 tier=production

$ kubectl label node $node3 tier=production

komutları çalıştırılmıştır. 


Wordpress uygulamasını hem "test" hem de "production" namespace'lerinde deploy edildi. (wordpress:latest ve mysql:5.6 imajlarından oluşturuldu)

mysql "ClusterIp" tipi bir servisle cluster içinden erişilebilir olacak.
Her iki uygulamanın da uzun dönem verilerini "persistent volume"ler üzerinde saklanacak.
Her iki uygulamanın da hiç bir hassas bilgisi "ör: şifre" uygulama ya da yaml dosyaları içerisinde tutulmayacak.
Her iki uygulama da aynı worker node üstünde schedule edilecek.
Her iki uygulama için de cpu ve memory kaynak kısıtları tanımlı olacak.

Bu işlemleri gerçekleştirmek için;

Prod ve test ortamları için secret oluşturma işlemi:

cd deployment

kubectl create secret generic mysql-test-secret -n test --from-file=MYSQL_ROOT_PASSWORD=./mysql_root_password.txt --from-file=MYSQL_USER=./mysql_user.txt --from-file=MYSQL_PASSWORD=./mysql_password.txt --from-file=MYSQL_DATABASE=./mysql_database.txt

kubectl create secret generic mysql-prod-secret -n production --from-file=MYSQL_ROOT_PASSWORD=./mysql_root_password.txt --from-file=MYSQL_USER=./mysql_user.txt --from-file=MYSQL_PASSWORD=./mysql_password.txt --from-file=MYSQL_DATABASE=./mysql_database.txt

yukarıdaki komutlar ile gerçekleştirilir.(Bu dosyaların txt ile verilmesinin amacı shell de bulunmaması.)

# PVC Nedir?

PersistentVolumeClaim (PVC), Kubernetes ortamında bir kullanıcının depolama kaynağı talep etmesini sağlayan bir nesnedir. Bu talep, Kubernetes tarafından otomatik olarak yönetilen PersistentVolume (PV) nesneleri ile eşleşir. PVC, kullanıcıların depolama ihtiyaçlarını belirtmelerine ve bu ihtiyaçlara uygun bir depolama birimi almalarına olanak tanır. Bu süreç, depolama kaynaklarının kullanıcılar ve uygulamalar için kolayca erişilebilir ve yönetilebilir olmasını sağlar.

## PVC Kullanımının Avantajları:
Kolay Yönetim: Kullanıcılar depolama kaynaklarını talep edebilir ve Kubernetes bu kaynakları otomatik olarak yönetir.
Kalıcı Depolama: Pod'lar yeniden başlatıldığında veya taşındığında veri kaybı olmaz.
Kaynak Paylaşımı: PVC, birden fazla Pod tarafından paylaşılabilir, böylece veriler merkezi bir depolama biriminde tutulur.
Dinamik Dağıtım: Kubernetes, dinamik olarak depolama kaynaklarını talep edebilir ve bu kaynakları kullanıcılara tahsis edebilir.



cd deployment 
komutu ile ilgili  klasöre geçilir.

kubectl apply -f .

komutu ile services, deployments ve persistentVolumes oluştururlur.

Sonrasında wordpress ui görmek için:
kubectl port-forward -n test deployment/wp-deployment 8080:80
kubectl port-forward -n production deployment/wp-deployment 8080:80

komutu ile port yönlendirmesi yapılır. Bu komut ile çalıştırdığımızda dış dünyayadan 8080 portuna gelen istekleri alıp 80 portuna yönlendirme işlemini ypapıyor.
Tarayıcıdan 127.0.0.1:8080 e gidilir ve wordpress web ui görülür.

Bunlar nginx ingressler oluşturulmadan önce yukarıda yapılan işlemlerin başarılı olup olmadığını görmek için yapılmıştır.


cd ingress-nginx/
klaösürüne geçilir.

kubectl apply -f ingress.yaml 

komutu ile ingressler test ve prod için oluşturulur.

kubectl get ingress -A -w
NAMESPACE    NAME              CLASS    HOSTS                 ADDRESS         PORTS   AGE
production   wp-prod-ingress   <none>   rumeysayuk.prod.com   192.168.112.2   80      100s
test         wp-test-ingress   <none>   rumeysayuk.test.com   192.168.112.2   80      100s
 
komutu ile görüntülenirler.

Ingresslerin çalışıp çalışmadığını görmek için domainleri host dosyamıza atarak deneyebiliriz.

linux için: sudo nano /etc/hosts

içine girilir ve ingress komutundan alınan address ve name alanları bu dosyaya eklenir.

daha sonra 
cd ..
ile ana dizine geçildi.

"production" namespace'inde "ozgurozturknet/k8s:v1" imajından, 5 replikalı, update stratejisi olarak aynı anda 2 pod'un update edilebileceği bir deployment oluşturuldu. "/healthcheck" endpoint'ini sorgulayan "liveness probe" ve "/ready" endpoint'ini sorgulayan bir "readiness probe" tanımları eklendi.

kubectl apply -f deployment.yaml 
ile deploy işlemi yapıldı.

Bir önceki adımda  oluşturulan deployment'i "loadbalancer" tipi bir servisle dış dünyadan erişilebilir hale getirildi.

kubectl expose deployment k8s-deployment --type=LoadBalancer -n production

Bu deployment'i önce 3 replikaya indirildi. Ardından 10 replikaya çıkarıldı. Sonrasında bu deployment'i "ozgurozturknet/k8s:v2" imajıyla güncellendi.


$ kubectl scale deployment k8s-deployment --replicas=3 -n production

$ kubectl scale deployment k8s-deployment --replicas=10 -n production

$ kubectl set image deployment/k8s-deployment k8s=ozgurozturknet/k8s:v2 -n production

