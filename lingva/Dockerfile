FROM thedaviddelta/lingva-translate:latest

ARG site_domain=localhost
ARG dark_theme=false
ARG default_source_lang=auto
ARG default_target_lang=en

ENV NEXT_PUBLIC_SITE_DOMAIN=$site_domain
ENV DEFAULT_DARK_THEME=$dark_theme
ENV NEXT_PUBLIC_DEFAULT_SOURCE_LANG=$default_source_lang
ENV NEXT_PUBLIC_DEFAULT_TARGET_LANG=$default_target_lang

RUN ["yarn", "build"]
CMD ["yarn", "start"]
