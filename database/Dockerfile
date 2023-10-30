# on utilise l'image postgres version 14.1
FROM postgres:14.1-alpine

# on initialise les variables de la db
ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

# on copie nos deux fichiers SQL de création de tables et de remplissage de la table
COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d/
COPY 02-InsertData.sql /docker-entrypoint-initdb.d/