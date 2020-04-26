FROM golang:1.14.2

# Install dependencies

ADD . /app

WORKDIR /app

RUN go get -d 

RUN go build

ENTRYPOINT [ "./go-rest-api" ]