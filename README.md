# Wordpress en 3 capas LEMP con Alta disponibilidad - Antonio Cruz Clavel - 13/12/2023
# Índice
# 1. Descripción del proyecto
Con esta práctica se quiere dar a conocer como implementar una pila LEMP en una infraestructura de tres capas y desplegar el CMS "Wordpress".  
La estructura sería la siguiente:  
### **Capa 1**:  
**Balanceador de carga nginx** con interfaz de red pública que distribuye las solicitudes entre los servers web llamada *BalanceadorAntonioCruz*  
  IP: 

### **Capa 2**: 
Backend con:  
- Dos máquinas con un servidor Nginx cada una llamadas *serverweb1AntonioCruz* y *serverweb2AntonioCruz*.
  IP: 
- Una máquina con un servidor NFS y motor PHP-FPM llamada *serverNFSAntonioCruz*.
  IP:

**Capa 3**: Máquina servidor de base de datos llamada *serverdatos1AntonioCruz*.  
 IP: 
Las capas 2 y 3 no están expuestas a red pública. Los servidores web utilizan una carpeta compartida por NFS desde el serverNFS y además utilizan el motor PHP-FPM instalado es una misma máquina.

# 2. Esquema del proyecto
# 3. Aprovisionamiento de las máquinas
