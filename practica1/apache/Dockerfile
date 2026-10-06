FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    apache2 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Desactivar prefork y activar MPM Event
RUN a2dismod mpm_prefork mpm_worker || true \
    && a2enmod mpm_event

COPY puertos.conf /etc/apache2/ports.conf
COPY sites-available/ /etc/apache2/sites-available/

RUN a2dissite 000-default.conf || true \
    && a2ensite 000-default.conf cliente.conf privado.conf

EXPOSE 8011

CMD ["apachectl", "-D", "FOREGROUND"]