FROM httpd

COPY ./src /usr/local/apache2/htdocs

RUN echo '<VirtualHost *:80> \n\
  ProxyPreserveHost On \n\
  ProxyPass /api/ balancer://opsapi \n\
  ProxyPassReverse /api/ balancer://opsapi \n\
  ProxyPass / http://front:80/ \n\
  ProxyPassReverse / http://front:80/ \n\
  <Proxy balancer://opsapi> \n\
  BalancerMember http://opsapi1:8080 route=backend1 \n\
  BalancerMember http://opsapi2:8080 route=backend2 \n\
  </Proxy> \n\
  </VirtualHost> \n\
  LoadModule proxy_module modules/mod_proxy.so \n\
  LoadModule proxy_http_module modules/mod_proxy_http.so \n\
  LoadModule slotmem_shm_module modules/mod_slotmem_shm.so \n\
  LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so \n\
  LoadModule proxy_balancer_module modules/mod_proxy_balancer.so \n\
  ' >> /usr/local/apache2/conf/httpd.conf