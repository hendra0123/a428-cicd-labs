# a428-cicd-labs
Latihan ini akan mengajarkan Anda tentang bagaimana cara menggunakan Jenkins untuk mengorkestrasi aplikasi React App. Aplikasi ini menggunakan React dan juga Node.js untuk menampilkan halaman website dengan konten “Welcome to React”, disertai dengan pengujian untuk memeriksa berjalannya proses rendering pada aplikasi. Adapun Jenkins disini berfungsi untuk memeriksa dan memastikan aplikasi React yang Anda buat tetap stabil dan siap digunakan. Sedangkan untuk Docker, memastikan bahwa Jenkins dan aplikasi React berjalan dengan cara yang konsisten dan efisien, tanpa bergantung pada konfigurasi komputer lokal. Setelah tahu akan hal itu, mari kita langsung saja buat apa yang harusnya kita lakukan.

Untuk mengikuti latihan ini, tidak harus menggunakan Operation System tertentu karena baik Linux, Windows, dan Mac bisa tetap dijalankan dan hanya sedikit perbedaan syntax (penggunaan perintah) saja, tapi pada kesempatan kali ini saya akan membahasnya menggunakan Windows OS. Selanjutnya ada beberapa hal juga yang harus Anda perhatikan terkait version dari beberapa aplikasi berikut agar lebih lancar saat pembuatannya, yaitu:
-	Docker 28.0.1
-	Git 2.43.0.windows.1
-	Visual Studio Code 1.98.2
Bagaimana, sudah siap? Kalau sudah, yuk kita mulai perjalanannya.
1.	Buka aplikasi Terminal pada komputer Anda.

2.	Buatlah sebuah bridge network di Docker menggunakan perintah berikut.

docker network create jenkins

3.	Setelah itu, Kita akan coba menjalankan aplikasi React App menggunakan Docker container didalam sebuah Docker container (lebih tepatnya di dalam Blue Ocean container–nanti akan dibahas). Praktik ini disebut dengan dind alias docker in docker. Jadi, silakan download dan jalankan docker:dind Docker image menggunakan perintah berikut.

docker run --name jenkins-docker --detach --privileged --network jenkins --network-alias docker --env DOCKER_TLS_CERTDIR=/certs --volume jenkins-docker-certs:/certs/client --volume jenkins-data:/var/jenkins_home --publish 2376:2376 --publish 3000:3000 --restart always docker:dind --storage-driver overlay2


Berikut adalah penjelasan dari perintah di atas.

--name: Menentukan nama Docker container yang akan digunakan untuk menjalankan image.

--detach: Menjalankan Docker container di background. Meski begitu, instance ini dapat dihentikan nanti dengan menjalankan perintah docker stop jenkins-docker.

--privileged: Menjalankan dind (docker in docker alias docker di dalam docker) saat ini memerlukan privileged access (akses istimewa) agar bisa berfungsi dengan baik.

--network jenkins: Ini berhubungan dengan network yang dibuat pada langkah sebelumnya.

--network-alias docker: Membuat Docker di dalam Docker container tersedia sebagai hostname docker di dalam jenkins network.

--env DOCKER_TLS_CERTDIR=/certs: Mengaktifkan penggunaan TLS di Docker server. Ini direkomendasikan karena kita menggunakan privileged container. Environment variable ini mengontrol root directory di mana Docker TLS certificates dikelola.

--volume jenkins-docker-certs:/certs/client: Memetakan direktori /certs/client di dalam container ke Docker volume bernama jenkins-docker-certs.

--volume jenkins-data:/var/jenkins_home: Memetakan direktori /var/jenkins_home di dalam container ke Docker volume bernama jenkins-data. Ini akan memungkinkan Docker container lain dikelola oleh Docker container’s Docker daemon ini untuk mount data dari Jenkins.

--publish 2376:2376: Mengekspos Docker daemon port pada mesin host (komputer Anda). Ini berguna untuk mengeksekusi Docker command (perintah Docker) pada mesin host (komputer Anda) dalam mengontrol inner Docker daemon.

--publish 3000:3000: Mengekspos port 3000 dari docker in docker container.

--restart always: memastikan container restart dan tetap menyala tidak hanya saat ada kegagalan tetapi juga setelah server yang digunakan juga restart.

docker:dind: Ini adalah image dari docker:dind itu sendiri. Image ini bisa diunduh sebelum dijalankan menggunakan perintah docker image pull docker:dind.

--storage-driver overlay2: Storage driver untuk Docker volume. Lihat halaman "Docker Storage drivers” untuk berbagai opsi yang didukung. 

Apakah Anda sudah mulai merasa cukup pusing melihat perintah Docker di atas? Tenang. Itu wajar dan tidak masalah jika Anda tidak begitu mahir dalam hal Docker karena fokus dari latihan ini adalah pada praktik continuous integration, bukan pada Docker. Untuk saat ini, Anda bisa jalankan sesuai perintah saja.

4.	Setelah melakukan langkah diatas, Docker container untuk Jenkins sudah berjalan. Namun, tugas kita belum selesai dan harus menjalankan Docker container lagi. Kali ini untuk Blue Ocean (UX terbaru dari Jenkins). Untuk itu, buatlah sebuah Dockerfile dengan perintah berikut.

echo. > Dockerfile

Catatan: Anda bisa menaruh Dockerfile di direktori yang Anda inginkan dengan pindah terlebih dahulu ke direktori tersebut menggunakan perintah cd path_directory. Misalnya, cd  Downloads.

5.	Selanjutnya salin kode berikut ke Dockerfile Anda.

FROM jenkins/jenkins:2.426.3-jdk11
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean:1.25.5 docker-workflow:1.28"

6.	Simpan file tersebut dengan menekan CTRL+S.
7.	Buat sebuah Docker image baru dari Dockerfile tadi dan berikan nama myjenkins-blueocean: 2.426.3-1.

docker build -t myjenkins-blueocean:2.426.3-1 .

Jangan lupa tanda (.) yang berada di akhir kode. Perintah di atas akan mengunduh official Jenkins Docker image secara otomatis.

8.	Setelah itu, jalankan myjenkins-blueocean:2.346.1-1 image sebagai container di Docker menggunakan perintah berikut.

docker run --name jenkins-blueocean --detach --network jenkins --env DOCKER_HOST=tcp://docker:2376 --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 --publish 8080:8080 --publish 50000:50000 --volume jenkins-data:/var/jenkins_home --volume jenkins-docker-certs:/certs/client:ro --volume "%USERPROFILE%:/home" --volume "D:/uctugas6/a428-cicd-labs:/home/a428-cicd-labs" --restart=on-failure --env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true" myjenkins-blueocean:2.426.3-1

Berikut adalah penjelasan dari perintah di atas.

--name: Menentukan nama Docker container yang akan digunakan untuk menjalankan image.

--detach: Menjalankan Docker container di background.

--network jenkins: Menghubungkan container ini dengan jenkins network yang dibuat sebelumnya. Ini membuat Docker daemon dari langkah sebelumnya tersedia ke container ini melalui hostname docker.

--env DOCKER_HOST=tcp://docker:2376: Menentukan environment variable yang digunakan oleh docker, docker-compose, dan Docker tools lainnya untuk terhubung ke Docker daemon dari langkah sebelumnya.

--publish 8080:8080: Memetakan (memublikasikan/mengekspos) port 8080 dari container saat ini ke port 8080 pada mesin host (komputer Anda). Angka pertama merepresentasikan port pada host, sedangkan yang terakhir mewakili port pada container. Jadi, jika Anda menentukan -p 49000:8080 untuk opsi ini, Anda akan mengakses Jenkins di mesin host melalui port 49000.

--publish 50000:50000: Memetakan (mengekspos) port 50000 dari container saat ini ke port 50000 pada mesin host (komputer Anda). Ini hanya diperlukan jika Anda telah menyiapkan satu atau lebih inbound Jenkins agents di mesin lain yang berinteraksi dengan jenkins-blueocean container Anda (Jenkins "controller"). Inbound Jenkins agents berkomunikasi dengan Jenkins controller melalui TCP port 50000 secara default. Anda dapat mengubah port number ini pada Jenkins controller melalui halaman Configure Global Security.

--volume jenkins-data:/var/jenkins_home: Memetakan direktori/var/jenkins_home pada container ke Docker volume dengan nama jenkins-data.

--volume jenkins-docker-certs:/certs/client:ro: Memetakan direktori/certs/client pada container ke volume yang dibuat sebelumnya yangbernama jenkins-docker-certs. Ini membuat client TLS certificates untukterhubung ke Docker daemon yang tersedia di path yang ditentukan olehDOCKER_CERT_PATH environment variable.

--volume "$HOME":/home: Memetakan direktori $HOME pada mesin host(komputer Anda, biasanya direktori /Users/<your-username>) ke direktori/home pada container.

--restart=on-failure: Mengonfigurasi Docker container restart policy agarmemulai kembali (restart) saat fail (terjadi kegagalan).

--env JAVA_OPTS="-Dhudson.plugins.git.GitSCM.ALLOW_LOCAL_CHECKOUT=true":Mengizinkan checkout pada local repository.

myjenkins-blueocean:2.346.1-1: Nama Docker image yang Anda buat dilangkah sebelumnya.


















 

