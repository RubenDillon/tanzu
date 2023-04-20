

    Este proyecto consta de 2 componentes, un loop y la ejecucion de chrome

    loop.sh
```
#!/bin/bash

while true; do
  # Ejecutar el batch aquí
  ./abrir2.sh
  
  # Esperar 20 minutos
  sleep 20
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
