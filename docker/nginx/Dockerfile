FROM nginx
WORKDIR /usr/share/nginx/html
COPY app/ . 
COPY config/nginx.conf /etc/nginx/nginx.conf
CMD ["nginx", "-g", "daemon off;"]
