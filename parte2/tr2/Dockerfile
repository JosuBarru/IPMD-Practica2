FROM apache/superset 
USER root 
RUN pip install impyla
USER superset 
ENV SUPERSET_SECRET_KEY=your_secret_key_here 
RUN superset fab create-admin \
               --username admin \
               --firstname Superset \
               --lastname Admin \
               --email admin@superset.com \
               --password admin 
RUN superset db upgrade 
RUN superset init