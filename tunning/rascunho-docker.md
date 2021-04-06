acessar o container SqlServer

```bash

docker container exec -u root -it sql /bin/bash
cd var/opt/mssql/data

# Verificar o tamanho do arquivo de de forma legivel
ls -laSh | grep 'nome do db'



```
