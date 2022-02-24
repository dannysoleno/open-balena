# openBalena on balena
> https://www.balena.io/open/docs/getting-started/

```sh
tld=balena.local

fleet=balena-nested


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

echo "cat /etc/docker.env; exit" \
  | balena ssh ${uuid}.local api \
  | grep ^SUPERUSER_ > .balena/env

source .balena/env

BALENARC_BALENA_URL=${balena_device_uuid}.${tld}

balena login --credentials \
  --email "${SUPERUSER_EMAIL}" \
  --password "${SUPERUSER_PASSWORD}"

balena devices && balena releases "${fleet}"

unset BALENARC_BALENA_URL

balena login

# https://github.com/pdcastro/ssh-uuid
ssh-uuid -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  --service balena-nested \
  ${balena_device_uuid}.balena \
  ./run-tests.sh

...
```
