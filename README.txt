## Dataset

This project uses the public `dvdrental` sample database (Sakila-derived),
freely available via postgresqltutorial.com or Neon's Postgres sample
datasets. The dump isn't checked into this repo — restore it locally with:

    pg_restore -d dvdrental dvdrental.tar

(or grab a fresh copy from the source above if you don't already have the tar)