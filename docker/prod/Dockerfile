FROM node:20 AS build

WORKDIR /build_ui_app
COPY ui/package.json ui/package-lock.json ui/tsconfig.json ./
RUN npm ci --no-audit --no-fund
COPY ui/ ./
RUN npm run build

FROM papermerge/tivoli:0.4.2 AS token_validator
FROM papermerge/auth-server:0.6.2 AS auth_server
FROM python:3.10-bookworm AS papermerge_core

# Space separated list of Tesseract language codes to install
# (English is always installed). Example: --build-arg OCR_LANGUAGES="deu fra spa"
ARG OCR_LANGUAGES="deu"

ENV CORE_APP=/core_app
ENV PAPERMERGE__DATABASE__URL=sqlite:////db/db.sqlite3
ENV DJANGO_SETTINGS_MODULE=config.settings
ENV PAPERMERGE__AUTH__USERNAME=admin
ENV PAPERMERGE__AUTH__EMAIL=admin@example.com
ENV PAPERMERGE__OCR__DEFAULT_LANGUAGE=deu

ENV VIRTUAL_ENV=${CORE_APP}/.venv

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    python3-dev \
    tesseract-ocr \
    $(for lang in ${OCR_LANGUAGES}; do printf 'tesseract-ocr-%s ' "$lang"; done) \
    curl \
    imagemagick \
    gcc \
    nginx \
    supervisor \
    poppler-utils \
    ghostscript \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

RUN pip install "poetry==1.8.5" roco==0.1.1

RUN python -m venv ${VIRTUAL_ENV}

COPY poetry.lock pyproject.toml README.md LICENSE ${CORE_APP}/

WORKDIR ${CORE_APP}
RUN poetry install --no-root -vvv

COPY docker/prod/run.bash /run.bash
COPY docker/prod/supervisord.conf /etc/
COPY docker/prod/nginx.conf /etc/nginx/nginx.conf
COPY docker/prod/logging.yml ${CORE_APP}/logging.yml
COPY ./papermerge ${CORE_APP}/papermerge/
COPY ./docker/prod/config ${CORE_APP}/config
COPY ./docker/prod/manage.py ${CORE_APP}/manage.py

COPY ./docker/prod/scripts/ /usr/bin/
# Strip Windows line endings (CRLF) in case the repo was cloned on Windows
RUN sed -i "s/\r$//" /run.bash /usr/bin/*.sh /etc/supervisord.conf /etc/nginx/nginx.conf && chmod +x /usr/bin/*.sh
RUN chmod +x /run.bash

COPY --from=token_validator /tivoli_app/ /tivoli_app
COPY --from=auth_server /auth_server_app/ /auth_server_app
COPY --from=auth_server /usr/share/nginx/html /usr/share/nginx/html/auth_server
COPY --from=build /build_ui_app/build/ /usr/share/nginx/html/ui

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=10s --start-period=180s --retries=5 \
    CMD curl -fsS -o /dev/null http://localhost/ || exit 1

ENTRYPOINT ["/run.bash"]
CMD ["server"]
