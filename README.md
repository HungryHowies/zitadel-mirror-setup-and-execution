### CockroachDB
curl https://binaries.cockroachdb.com/cockroach-v23.2.7.linux-amd64.tgz  | tar -zx && sudo cp -i cockroach-v23.2.7.linux-amd64/cockroach /usr/local/bi
sudo mkdir -p /usr/local/lib/cockroach     
sudo cp -i cockroach-v23.2.7.linux-amd64/lib/libgeos.so /usr/local/lib/cockroach/
sudo cp -i cockroach-v23.2.7.linux-amd64/lib/libgeos_c.so /usr/local/lib/cockroach/
which cockroach
cockroach start-single-node --insecure --background --http-addr :9000 --listen-addr=localhost

### Zitadel
wget https://github.com/zitadel/zitadel/releases/download/v2.54.6/zitadel-linux-amd64.tar.gz
tar -zxvf zitadel-linux-amd64.tar.gz 
sudo mv zitadel-linux-amd64/zitadel /usr/local/bin/
cd /usr/local/bin/
vi defaults.yaml

### Let's encrypt
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx
vi  /etc/nginx/sites-available/zitadel
cd /etc/nginx/sites-enabled
rm default
ln -s /etc/nginx/sites-available/zitadel  /etc/nginx/sites-enabled/zitadel
nginx -t


### Zitadel Default Configuration file
cd /usr/local/bin/
vi defaults.yaml 
zitadel start-from-init   --config defaults.yaml  --masterkey "MasterkeyNeedsToHave32Characters"  --tlsMode external

### Zitadel Service
vi /etc/systemd/system/zitadel.service
systemctl daemon-reload

## No Isssue I proceed to Migration Mirror from CockroachDB to Postgresql
	 
### Postgres Install
sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
apt  update
apt -y install postgresql

## Create Zitadel Database
sudo -u postgres psql
CREATE ROLE zitadel LOGIN;
CREATE DATABASE zitadel;
GRANT CONNECT, CREATE ON DATABASE zitadel TO zitadel;

### MMirror  Configuration file
vi config.yaml
