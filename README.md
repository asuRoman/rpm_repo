#Сборка RPM-пакета и создание репозитория

#Задание:

#создать свой RPM (можно взять свое приложение, либо собрать к примеру Apache с определенными опциями);

#cоздать свой репозиторий и разместить там ранее собранный RPM;

#реализовать это все либо в Vagrant, либо развернуть у себя через Nginx и дать ссылку на репозиторий.

#установить пакеты для создания своего репозитория

 yum install -y wget rpmdevtools rpm-build createrepo \ yum-utils cmake gcc git nano

#создадим директорию и загрузим пакет SPRM NGINX для дальнейшего его изменения

mkdir rpm && cd rpm

yumdownloader --source nginx

#ставим пакеты и зависимости для сборки этого пакета

rpm -Uvh nginx*.src.rpm

yum-builddep nginx

#необходимо скачать исходный код модуля ngx_brotli

cd /root

git clone --recurse-submodules -j8 \ https://github.com/google/ngx_brotli

cd ngx_brotli/deps/brotli

mkdir out && cd out


#дальше собираем данный модуль

cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..

cmake --build . --config Release -j 2 --target brotlienc

cd ../../../..

#собираем RPM пакет

cd ~/rpmbuild/SPECS/

rpmbuild -ba nginx.spec -D 'debug_package %{nil}'

#копируем пакеты в каталог

cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/

cd ~/rpmbuild/RPMS/x86_64

#устанаввливаем и запускаем измененный пакет

yum localinstall *.rpm

systemctl enable --now nginx

#создаем директорию для своего репозитория

mkdir /usr/share/nginx/html/repo_asur

cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo_asur/

#иницилизируем репозиторий

createrepo /usr/share/nginx/html/repo_asur/

#перезапускаем NGINX

nginx -t

nginx -s reload

#для теста репозиторя добавим его в конф файл

cat >> /etc/yum.repos.d/otus.repo_asur << EOF

[asur]

name=asur-linux

baseurl=http://localhost/repo_asur

gpgcheck=0

enabled=1

EOF


#добавляем наш измененный пакет в этот рпеозиторий

cd /usr/share/nginx/html/repo/

#обновим список пакетов в репозитории

createrepo /usr/share/nginx/html/repo_asur/

yum makecache
