FROM docker.io/library/alpine:3.19.1 as base
ARG TARGETPLATFORM
ENV PATH="${PATH}:/root/.krew/bin"

RUN apk add --no-cache bash ca-certificates catatonit curl git jq util-linux yq

COPY --from=ghcr.io/fluxcd/flux-cli:v2.3.0 /usr/local/bin/flux /usr/local/bin/flux
COPY --from=ghcr.io/siderolabs/talosctl:v1.7.3 /talosctl /usr/local/bin/talosctl
COPY --from=quay.io/prometheus/alertmanager:v0.27.0 /bin/amtool /usr/local/bin/amtool
COPY --from=registry.k8s.io/kubectl:v1.30.1 /bin/kubectl /usr/local/bin/kubectl

RUN curl -fsSL "https://i.jpillora.com/kubernetes-sigs/krew!!?as=krew&type=script" | bash
RUN krew install cnpg && kubectl cnpg version


COPY ./apps/fstrim/entrypoint.sh /entrypoint.sh

WORKDIR /app

ENTRYPOINT ["/usr/bin/catatonit", "--"]
CMD ["/entrypoint.sh"]
