FROM nginx:alpine

# Se copian los ficheros dentro de la imagen (no se montan como volumen) porque
# las ACL del volumen de UGOS impiden que el usuario `nginx` los lea: daba 403.
COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf
COPY web/index.html /usr/share/nginx/html/index.html

RUN chmod -R a+rX /usr/share/nginx/html
