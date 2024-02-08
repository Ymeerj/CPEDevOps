FROM httpd

COPY ./src /usr/local/apache2/htdocs

RUN echo '<VirtualHost *:80> \n\
  ProxyPreserveHost On \n\
  ProxyPass /api/ balancer://opsapi \n\
  ProxyPassReverse /api/ balancer://opsapi \n\
  ProxyPass / http://front:80/ \n\
  ProxyPassReverse / http://front:80/ \n\
  </VirtualHost> \n\
  <Proxy balancer://opsapi> \n\
  BalancerMember http://ospapi1:8080 route=backend1 \n\
  BalancerMember http://posapi2:8080 route=backend2 \n\
  </Proxy> \n\
  LoadModule proxy_module modules/mod_proxy.so \n\
  LoadModule proxy_http_module modules/mod_proxy_http.so \n\
  LoadModule proxy_balancer_module modules/mod_proxy_balancer.so \n\
  ' >> /usr/local/apache2/conf/httpd.conf