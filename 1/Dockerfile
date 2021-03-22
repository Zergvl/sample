FROM ecwid/ops-test-task:20210311a
RUN  sed -i -r "s/8000/8082/g" /etc/nginx/sites-enabled/box.conf &&  sed -i -r "s/-Xmx50m/-Xmx512m/g" /etc/init.d/box &&  sed -i -r "s/peer/trust/g" /etc/postgresql/12/main/pg_hba.conf && service postgresql stop && service postgresql start
COPY dump /opt/box/
RUN service postgresql start &&  /usr/bin/psql -U postgres < /opt/box/dump

EXPOSE 80/tcp
CMD ["/startup"]