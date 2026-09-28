FROM golang:1.21-alpine
WORKDIR /app
RUN apk add --no-cache git
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -v -o auth-api
EXPOSE 8000
CMD ["./auth-api"]