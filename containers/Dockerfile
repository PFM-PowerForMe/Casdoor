FROM docker.io/library/alpine:edge AS git
ARG TAG
ENV TAG=${TAG:-main}
RUN apk update && apk add --no-cache \
	git
WORKDIR /
RUN git -c advice.detachedHead=false clone --recurse-submodules https://github.com/pfm-powerforme/casdoor.git source
WORKDIR /source/
RUN git -c advice.detachedHead=false checkout $TAG


FROM docker.io/library/node:lts-alpine AS frontend
RUN apk update && apk add --no-cache \
	gcc musl-dev g++ make py3-pip
RUN corepack enable
ENV CYPRESS_DOWNLOAD_MIRROR="https://download.cypress.io"
WORKDIR /
COPY --from=git /source/web/ /frontend/
WORKDIR /frontend/
RUN mkdir -p /frontend/.yarn/cache \
    && mkdir -p /frontend/.yarn/berry/cache
RUN --mount=type=cache,target=/frontend/.yarn/cache \
    --mount=type=cache,target=/frontend/.yarn/berry/cache \
    BUILD_SEQUENCIAL=1 yarn config set cacheFolder /frontend/.yarn/cache
RUN yarn install --frozen-lockfile
RUN yarn run build


FROM docker.io/library/golang:alpine AS backend
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux
RUN apk update && apk add --no-cache \
	git
WORKDIR /
COPY --from=git /source/ backend/
WORKDIR /backend/
RUN go build -ldflags="-w -s" -o server
RUN go test -v -run TestGetVersionInfo util/system_test.go util/system.go > version_info.txt


FROM docker.io/library/alpine:latest AS runtime
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
COPY --from=backend --chown=$USER:$USER /backend/server server
COPY --from=backend --chown=$USER:$USER /backend/conf/app.conf conf/app.conf
COPY --from=backend --chown=$USER:$USER /backend/version_info.txt backend/version_info.txt
COPY --from=backend --chown=$USER:$USER /backend/swagger/ swagger/
COPY --from=frontend --chown=$USER:$USER /frontend/build/ web/build/
ENTRYPOINT ["/server"]
