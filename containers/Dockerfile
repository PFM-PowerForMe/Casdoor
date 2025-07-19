FROM alpine:edge AS git
ARG TAG
ENV TAG=${TAG:-main}
RUN apk update && apk add --no-cache \
	git
WORKDIR /tmp/
RUN git -c advice.detachedHead=false clone --recurse-submodules https://github.com/pfm-powerforme/casdoor.git casdoor
WORKDIR /tmp/casdoor/
RUN git -c advice.detachedHead=false checkout $TAG


FROM node:22-alpine AS frontend
WORKDIR /
COPY --from=git /tmp/casdoor/web/ web/
WORKDIR /web/
RUN yarn install --frozen-lockfile --network-timeout 1000000 && NODE_OPTIONS="--max-old-space-size=4096" yarn run build


FROM golang:1-alpine AS backend
RUN apk update && apk add --no-cache \
	git
WORKDIR /go/src/
COPY --from=git /tmp/casdoor/ casdoor/
WORKDIR /go/src/casdoor/
RUN CGO_ENABLED=0 go build -ldflags="-w -s" -o server
RUN go test -v -run TestGetVersionInfo util/system_test.go util/system.go > version_info.txt


FROM alpine:latest AS runtime
ARG USER=casdoor
RUN apk update && apk add --no-cache \
  sudo \
  tzdata \
  curl \
  ca-certificates
RUN update-ca-certificates
RUN adduser -D $USER -u 1000 \
    && echo "$USER ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/$USER \
    && chmod 0440 /etc/sudoers.d/$USER \
    && mkdir logs \
    && chown -R $USER:$USER logs
USER 1000
WORKDIR /
COPY --from=backend --chown=$USER:$USER /go/src/casdoor/server server
COPY --from=backend --chown=$USER:$USER /go/src/casdoor/conf/app.conf conf/app.conf
COPY --from=backend --chown=$USER:$USER /go/src/casdoor/version_info.txt go/src/casdoor/version_info.txt
COPY --from=backend --chown=$USER:$USER /go/src/casdoor/swagger/ swagger/
COPY --from=frontend --chown=$USER:$USER /web/build/ web/build/
ENTRYPOINT ["server"]
