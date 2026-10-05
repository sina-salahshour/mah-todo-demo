FROM debian:trixie-slim

WORKDIR /app

COPY ./out/todo.mahc /app/todo

RUN chmod +x /app/todo

EXPOSE 3000

CMD ["/app/todo", "--host", "0.0.0.0", "--port", "3000"]
