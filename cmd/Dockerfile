# Build the manager binary
FROM golang:1.16 as builder

WORKDIR /workspace

ARG APISERVER_NETWORK_PROXY_VERSION=0.0.27
ARG ADDON_AGENT_IMAGE_NAME

# Build Apiserver-network-proxy binaries
RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 \
      wget https://github.com/kubernetes-sigs/apiserver-network-proxy/archive/refs/tags/v${APISERVER_NETWORK_PROXY_VERSION}.tar.gz \
      && tar xzvf v${APISERVER_NETWORK_PROXY_VERSION}.tar.gz \
      && cd apiserver-network-proxy-${APISERVER_NETWORK_PROXY_VERSION} \
      && go build -o /workspace/proxy-server ./cmd/server/ \
      && go build -o /workspace/proxy-agent ./cmd/agent/ \
      && go build -o /workspace/proxy-test-client ./cmd/client/ \
      && go build -o /workspace/proxy-test-server ./cmd/test-server/ \
      && cd /workspace

# Copy the Go Modules manifests
COPY go.mod go.mod
COPY go.sum go.sum
# cache deps before building and copying source so that we don't need to re-download as much
# and so that source changes don't invalidate our downloaded layer
RUN GOPROXY="https://goproxy.cn/,direct" go mod download

# Copy the go source
COPY cmd/ cmd/
COPY pkg pkg/
COPY controllers controllers/


# Build addons
RUN GOPROXY="https://goproxy.cn/,direct" CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o agent cmd/addon-agent/main.go
RUN GOPROXY="https://goproxy.cn/,direct" CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a \
        -ldflags="-X 'open-cluster-management.io/cluster-proxy/pkg/config.AgentImageName=${ADDON_AGENT_IMAGE_NAME}'" \
        -o manager cmd/addon-manager/main.go

# Use distroless as minimal base image to package the manager binary
# Refer to https://github.com/GoogleContainerTools/distroless for more details
FROM alpine:3.13

WORKDIR /
RUN apk add libc6-compat
COPY --from=builder /workspace/proxy-server .
COPY --from=builder /workspace/proxy-agent .
COPY --from=builder /workspace/proxy-test-client .
COPY --from=builder /workspace/proxy-test-server .
COPY --from=builder /workspace/agent .
COPY --from=builder /workspace/manager .
USER 65532:65532
