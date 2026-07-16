FROM alpine:3.20 AS base
RUN echo base > /stage.txt

FROM base AS builder
RUN echo builder > /stage.txt

FROM base AS test
RUN echo test > /stage.txt

FROM alpine:3.20
COPY --from=builder /stage.txt /stage.txt
CMD ["cat", "/stage.txt"]
