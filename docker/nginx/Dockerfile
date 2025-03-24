FROM nginx:1.26.3
WORKDIR /usr/share/nginx/html
COPY app/ . 
COPY config/nginx.conf /etc/nginx/nginx.conf
CMD ["nginx", "-g", "daemon off;"]
