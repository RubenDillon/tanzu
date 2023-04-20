
Este proyecto consta de 2 componentes, 
- el script loop.sh
- la ejecucion de chrome desde abrir.sh

loop.sh
```
#!/bin/bash

while true; do
  # Ejecutar el batch aquí
  ./abrir.sh
  
  # Esperar 20 minutos
  sleep 1200
done
```

abrir.sh

```
fecha_actual=$(date '+%Y-%m-%d %H:%M:%S')
echo "La fecha y hora actual es: $fecha_actual"


google-chrome-stable https://www.youtube.com/watch?v=z4CPeE2B4NE&list=PLBrdJqPHEZjtLZCfYeB4w5HnFWv3mcmaT

```


Para poner en hora la VM
```
    sudo timedatectl set-timezone America/Argentina/Buenos_Aires
```
