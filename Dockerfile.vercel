FROM debian:trixie-slim

WORKDIR /app

COPY ./out/todo.mahc /app/todo

RUN chmod +x /app/todo

CMD ["/bin/sh", "-c", "/app/todo --host 0.0.0.0 --port ${PORT}"]
