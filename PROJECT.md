# 1 run node docker
 ```
 docker run -it --privileged [image]
 docker ps -a   // show the vm id
 docker cp [my_binary] vm-id:binary-path
 docker commit vm-id [image]
 ```