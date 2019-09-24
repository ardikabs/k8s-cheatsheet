# Kubernetes Cheatsheet

### Nodes Problem
1. **Docker hang**
    Biasa terjadi waktu workload sedang tinggi banget atau karena sudah kerja terlalu lama.
    Solusinya _take-out_ dari cluster, lalu _clean-up_ data docker
    Command:
    - `kubectl cordon $NODE_NAME`
    - `kubectl drain $NODE_NAME --ignore-daemonsets --delete-local-data [--force]`
    - `mv /var/lib/docker/overlay2 /var/lib/docker/overlay2.old`
    - `rsync --delete /dev/null /var/lib/docker/overlay2.old`
    - `mv /var/lib/docker/overlay2.old /tmp/overlay2.old`
    - `reboot`

2. **Hardware failure**
    Masalahnya berkaitan dengan permasalahan dari team DCO, langkah - langkah tetap sama sesuai dengan solusi poin 1, namun tanpa menghapus storage data.

3. **BGP down**
    Sebenarnya permasalahannya berkaitan dengan **Docker hang** karena BGP client kita hidup sebagai standalone docker container. Jadi langkah terbaiknya mengikuti solusi poin 1

### Microservices Problem
1. **CrashLoopBackOff**
    Arti sederhananya, berarti kubernetes dalam mencoba membangun pod selalui menemui kegagalan untuk mencapai state live dari si pod tersebut. Masalahnya bisa jadi karena liveness timeout, workload pod terlalu tinggi, images, aplikasi gagal up (codes problem) dll.
    Langkah troubleshoot:
    - `kubectl describe pod $POD_NAME` -> untuk check liveness probe
    - `kubectl logs $POD_NAME` -> untuk check workload pod terlalu tinggi / code problem

2. **Increased Failure Rate (5xx) between mServices**
    Ini bisa diasumsikan karena hop nya bertambah, seharusnya semua service yang sudah ada di kubernetes itu kalau berkomunikasi harus melalui kubernetes itu sendiri (dns, serviceIP), sedangkan di kubernetes cluster 3 (production) untuk berkomunikasi antar service melalui NodePort, padahal tujuan dari NodePort untuk komunikasi kepada eksternal atau service yang tidak ada di kubernetes cluster.

3. **Perhatikan label Pod**
    Biasanya, pada deployment yang digunakan sekarang, seluruh Pod punya label yang sama meskipun dalam High-level resources yang berbeda (Deployment, CronJob). Dikarenakan, pattern yang sama juga dipakai untuk Services dalam menentukan Pod mana yang harus masuk kedalam ip pool dari Service yang didefinisikan, hal ini menyebabkan Service berekspektasi Pod memiliki Port yang terbuka, sedangkan seringkalinya Pod dari CronJob tidak mendefinikan port yang terbuka atau bahkan images pada Pod tersebut default-nya tidak membuka port sama sekali, akhirnya pada saat mengakses Service IP tersebut, akan sering menemui masalah tidak dapat melakukan `kubectl port-forward` dan/atau mengalami error 5xx.

### Access Problem
1. https://kubernetes-svc-3.bukalapak.io:6443
    Harus melakukan request token kepada KubeLB tim. Lalu tim melakukan pembuatan token sesuai dengan RBAC yang dibutuhkan.
    Ingat!
    - Role (**Namespace-scope**) dan ClusterRole (**Cluster-wide**)
    - Role/ClusterRole mendefinisikan dia dapat melakukan apa
    - RoleBinding/ClusterRoleBinding mendefinisikan role/clusterrole berpasangan dengan siapa (ServiceAccount)
    - Role -> RoleBinding, ClusterRole -> RoleBinding/ClusterRoleBinding
2. Validate=true
    Beberapa hari ini command `kubectl apply -f xx.yaml`, `kubectl edit deploy $DEPLOYMENT_NAME` tidak dapat digunakan karena terjadi kegagalan pada proses validasi, solusi nya dengan menambahkan `--validate=false` atau memperbarui `kubectl` ke versi terbaru setidaknya versi 1.13

### General Design
* Kubernetes Cluster 3 punya 3 jenis nodes, yang dibedakan berdasarkan label environment. Diantaranya:
  - production
  - prometheus
  - gitlab
  - proxy
  - test
  Command:
  - `kubectl get node -l env=$ENV`
* Nodes dengan env=proxy difungsikan khusus sebagai proxy/endpoint dari eksternal menuju kubernetes. Berlaku untuk semua cluster (3,4,5)
* Shortcut untuk tau cara kondisi node
  - `kubectl get node -l env=production --no-headers | grep -iE "NotReady|Ready,SchedulingDisabled"`
* Menggunakan **calico** sebagai CNI (Container Network Interface)

### Additional Cheat
* `kubectl --all-namespaces get pod --field-selector status.podIP=$POD_IP`
* `kubectl get po -n ${NAMESPACE} --sort-by status.creationTimestamp`
* `kubectl get po -n ${NAMESPACE} --field-selector status.phase!=Running`
* `kubectl get po -n default -l project=${MICRO_SERVICE_NAME}` -> di kubernetes cluster 3
