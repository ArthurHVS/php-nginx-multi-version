# Rodando duas versões do PHP na mesma instância do NGINX

## Instalação:
São pré-requisitos:

### NGINX
    No Ubuntu:
        $ sudo apt-get update
        $ sudo apt-get install nginx

    Para gerenciar o nginx, use o systemd, por meio de sua ferramenta principal, o systemctl (exemplos servem para qualquer .service do sistema):
        Iniciar, parar e reiniciar o serviço:
            # systemctl start nginx.service
            # systemctl stop nginx.service
            # systemctl restart nginx.service
        Verificar status e se o serviço está habilitado no boot:
            # systemctl status nginx.service
            # systemctl is-enabled nginx.service
        Habilitar e desabilitar o serviço no boot
            # systemctl enable nginx.service
            # systemctl disable nginx.service

### Versões do PHP
    - As versões do fastcgi cabíveis aos projetos, nesse caso, 5.6 e 7.4
        Para instalar a versão 7.4 no Ubuntu:
            $ sudo apt update
            $ sudo apt install php php-cli php-fpm php-json php-mysql php-zip php-gd  php-mbstring php-curl php-xml php-pear php-bcmath
        
        Para instalar a versão 5.6 no Ubuntu:
            $ sudo apt install python-software-properties
            $ sudo add-apt-repository ppa:ondrej/php
            $ sudo apt-get install php5.6-fpm

### Ajustes de Firewall
Pode ser necessário ajustar as regras do firewall no servidor:
    No Ubuntu:
        $ sudo ufw app list
            (Output)
            Available Applications:
                Nginx Full
                Nginx HTTP
                Nginx HTTPS
                OpenSSH

    Você pode permitir cada uma dessas aplicações com:
        $ sudo ufw allow 'NginxHTTP'
    
    E pode verificar o status do firewall com:
        $ sudo ufw status
            (Output)
            Status: active

            To          Actions         From
            --          ------          ----
            OpenSSH     ALLOW           Anywhere
            Nginx HTTP  ALLOW           Anywhere
            [...]
        
    
## Páginas-exemplo PHP
    As pastas 'seven-four' e 'five-six' contém, cada uma, um arquivo index.php, com uma chamada ao método phpinfo(). É um jeito simples de mostrar a versão do php que aquela página está consumindo.

## Configurando o NGINX:
    Após a instalação do NGINX, a seguinte árvore de diretórios é criada dentro de /etc/nginx (Ubuntu/Debian):
        -/conf.d
        -/modules-available
        -/modules-enabled
        -/sites-available
        -/sites-enabled
        -/snippets

    Também são criados diversos arquivos, sendo os mais importantes:
        [...]
        nginx.conf
        fastcgi.conf
    
    A diferença entre as pastas /*-available e /*-enabled é que, nas primeiras, disponibilizamos os arquivos de configuração e nas últimas, os habilitamos de fato, criando links simbólicos para os arquivos da primeira. O arquivo nginx.conf contém as configurações para esses diretórios.
    
    A pasta /snippets contém alguns códigos padrão e pode ser extendida, caso haja necessidade. Para esse exemplo, basta o código já presente.

    A princípio, esses arquivos não precisam ser modificados, mas caso alguma variável de instalação do php apresente problemas, é nesses arquivos que o nginx faz as conexões entre as variáveis de sistema e as suas próprias.

    Sempre que qualquer arquivo for alterado, é necessário reiniciar o serviço do NGINX para que essas alterações surtam efeito. Claro que em produção não é interessante reiniciar o servidor sem ter certeza de que os arquivos de configuração estão, pelo menos, com a sintaxe correta. Uma maneira de testar essa sintaxe é:
        
        $ sudo nginx -t

### sites-available:
    Aqui, ficam configurados os 'sites' (locais, podendo ser páginas estáticas ou aplicações, como API's, CGI's, etc...) que o nginx tem DISPONÍVEIS para servir. Veja que só por estarem configurados aqui, eles ainda não estão HABILITADOS, sendo necessário criar um link simbólico para cada site que será habilitado, na pasta /sites-enabled

    Segue um bloco básico de configuração:

        server {
            listen 1234;
            listen [::]:1234;
            root /var/www/html/php-specific-site;
            index index.html index.php;
            location ~ \.php$ {
                
                include snippets/fastcgi-php.conf;
                include fastcgi_params;
                
                fastcgi_pass unix:/var/run/php/php5.6-fpm.sock;
                
                fastcgi_param SCRIPT_FILENAME /var/www/html php-specific-site$fastcgi_script_name;
            }
        }
    
    O servidor escuta requisições na porta 1234, redireciona para a pasta em que está o arquivo index.php e usa a fastcgi que configuramos (nesse caso, 5.6) parâmetro, para todos os arquivos .php disponíveis, sempre incluindo o snippet padrão (fastcgi-php.conf).

    Nos exemplos apresentados, o servidor ouve a porta 5600 para a versão 5.6 e a 7400 para a versão 7.4 do PHP. Como foi utilizado o Ubuntu, a fastcgi do php sempre ouvirá requisições em um socket unix, criado em /var/run/php/php*.*-fpm.sock. Caso o servidor utilize o protocolo TCP comum para esse socket (como no caso do Windows e algumas versões do Linux), o parâmetro fastcgi_pass deve ser modificado:

        {
            ...
            fastcgi_pass 127.0.0.1:9000;
            ...
        }
### sites-enabled:
    Aqui, ficam os links para as configurações disponíveis. Em sistemas como o Linux e o MacOS, fazemos:
    
    # ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/

    Observe que essa lógica também é válida para os módulos do nginx (configurações customizadas e plugins de terceiros, nas pastas modules-available e modules-enabled)