# openBalena on balena
> https://www.balena.io/open/docs/getting-started/

```sh
tld=balena.local

fleet=foo-bar

device_type=fincm3

os_version=2.83.21+rev1.dev


mkdir .balena

uuid=$(printf "results:\n$(sudo balena scan)" \
  | yq e '.results[] | select(.osVariant=="development").host' - \
  | awk -F'.' '{print $1}' | head -n 1) \
  && balena_device_uuid=$(balena device ${uuid:0:7} | grep UUID | cut -c24-)

balena push ${uuid}.local --nocache

cert_manager=$(DOCKER_HOST=${uuid}.local docker ps \
  --filter "name=cert-manager" \
  --format "{{.ID}}")

DOCKER_HOST=${uuid}.local docker cp \
  ${cert_manager}:/certs/private/ca-bundle.${balena_device_uuid}.${tld}.pem .balena/

export NODE_EXTRA_CA_CERTS="$(pwd)/.balena/ca-bundle.${balena_device_uuid}.${tld}.pem"

sudo security add-trusted-cert -d \
  -r trustAsRoot \
  -k /Library/Keychains/System.keychain \
  ${NODE_EXTRA_CA_CERTS}

echo "cat /balena/${balena_device_uuid}.${tld}.env; exit" \
  | balena ssh ${uuid}.local api \
  | grep SUPERUSER_PASSWORD > .balena/env

echo "cat /etc/docker.env; exit" \
  | balena ssh ${uuid}.local api \
  | grep SUPERUSER_EMAIL >> .balena/env

source .balena/env

BALENARC_BALENA_URL=${balena_device_uuid}.${tld}

balena login --credentials \
  --email "${SUPERUSER_EMAIL}" \
  --password "${SUPERUSER_PASSWORD}"

balena fleet create "${fleet}" --type "${device_type}"

wget -O .balena/balena.zip \
  "https://api.balena-cloud.com/download?deviceType=${device_type}&version=${os_version}&fileType=.zip&developmentMode=true"

unzip "$(find .balena -type f -name *.zip)" -d .balena/

image="$(find .balena -type f -name *.img)"

balena os configure "${image}" --fleet "${fleet}"

sudo balena local flash "${image}"

balena devices

printf 'FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine\nCMD [ "balena-idle" ]' \
  | balena deploy "${fleet}"--logs

balena releases "${fleet}"

balena device "$(balena devices --json | jq -r .[].uuid)"

balena tunnel "$(balena devices --json | jq -r .[].uuid)" -p 22222:2222 &

ssh root@localhost -p 2222
...
```
