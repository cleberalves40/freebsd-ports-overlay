PORTNAME=	thinker
DISTVERSIONPREFIX=	v
DISTVERSION=	0.41.2
CATEGORIES=	www
MASTER_SITES=	https://github.com/cleberalves40/freebsd-ports-overlay/releases/download/thinker-npm-cache/:npmcache
DISTFILES=	${NPM_CACHE_DIST}:npmcache

MAINTAINER=	cleber@bsd.com.br
COMMENT=	Open-source LLM gateway that routes agents to models and tracks cost
WWW=		https://github.com/patrickbrandao/thinker

LICENSE=	MIT
LICENSE_FILE=	${WRKSRC}/LICENSE

ONLY_FOR_ARCHS=	aarch64 amd64
ONLY_FOR_ARCHS_REASON=	build tools (rolldown, lightningcss, tailwind oxide) ship prebuilt binaries only for these

BUILD_DEPENDS=	npm${NODEJS_SUFFIX}>0:www/npm${NODEJS_SUFFIX}
RUN_DEPENDS=	postgresql18-server>0:databases/postgresql18-server

USES=		nodejs:24,build,run
USE_GITHUB=	yes
GH_ACCOUNT=	patrickbrandao
USE_RC_SUBR=	thinker thinker_db

SUB_FILES=	pkg-message thinker-cli thinker-run
SUB_LIST=	APPDIR=${APPDIR} THINKER_USER=${USERS} DBDIR=/var/db/thinker

USERS=		thinker
GROUPS=		thinker
UID_FILES=	${FILESDIR}/UIDs
GID_FILES=	${FILESDIR}/GIDs

NO_ARCH=	yes

APPDIR=		${PREFIX}/libexec/${PORTNAME}
NPM_CACHE_DIST=	${PORTNAME}-npm-cache-${DISTVERSION}.tar.xz
NPM_ENV=	HOME=${WRKDIR} npm_config_cache=${WRKDIR}/npm-cache \
		npm_config_offline=true npm_config_audit=false \
		npm_config_fund=false npm_config_update_notifier=false
NPM_CMD=	${SETENV} ${MAKE_ENV} ${NPM_ENV} ${LOCALBASE}/bin/npm
# Ordem de turbo.json (dependsOn ^build); o turbo nao tem binario FreeBSD
WORKSPACES=	thinker-shared thinker-backend thinker-frontend thinker-cli
# Arvore de producao, como o estagio prod-deps do Dockerfile oficial
PRODDIR=	${WRKDIR}/prod

do-build:
	cd ${WRKSRC} && ${NPM_CMD} ci --ignore-scripts --loglevel=error
.for w in ${WORKSPACES}
	cd ${WRKSRC} && ${NPM_CMD} run build -w ${w}
.endfor
	${MKDIR} ${PRODDIR}
	cd ${WRKSRC} && ${CP} package.json package-lock.json ${PRODDIR}/
.for p in shared frontend backend
	${MKDIR} ${PRODDIR}/packages/${p}
	${CP} ${WRKSRC}/packages/${p}/package.json ${PRODDIR}/packages/${p}/
.endfor
	cd ${PRODDIR} && ${NPM_CMD} ci --omit=dev --omit=optional \
		--ignore-scripts --loglevel=error
	cd ${PRODDIR} && ${SETENV} ${MAKE_ENV} ${LOCALBASE}/bin/node \
		${WRKSRC}/docker/third-party-notices.mjs ${PRODDIR} \
		${PRODDIR}/THIRD_PARTY_NOTICES
	cd ${PRODDIR}/node_modules && ${RM} -r typescript @typescript @types \
		ts-node acorn create-require v8-compile-cache-lib
	${FIND} ${PRODDIR}/node_modules \( -name '*.map' -o -name 'README*' \
		-o -name 'CHANGELOG*' -o -name '*.md' -o -name '*.gyp' \
		-o -name '*.gypi' -o \( -name '*.ts' ! -name '*.d.ts' \) \) \
		-type f -delete
	${FIND} ${PRODDIR}/node_modules -type d \( -name __tests__ -o -name test \
		-o -name tests -o -name example -o -name examples -o -name .github \) \
		-prune -exec ${RM} -r {} +
	${FIND} ${PRODDIR}/node_modules -type d -empty -delete
	@if ${FIND} ${PRODDIR}/node_modules \( -name '*.node' -o -name '*.so*' \) \
	    -type f | ${GREP} -q .; then \
		${ECHO_MSG} "===> runtime tree has native modules; NO_ARCH no longer holds"; \
		${FALSE}; \
	fi

do-install:
	${MKDIR} ${STAGEDIR}${APPDIR}/packages
	cd ${PRODDIR} && ${COPYTREE_SHARE} "node_modules package.json THIRD_PARTY_NOTICES" \
		${STAGEDIR}${APPDIR}
	${INSTALL_DATA} ${WRKSRC}/LICENSE ${STAGEDIR}${APPDIR}
.for p in shared backend frontend cli
	${MKDIR} ${STAGEDIR}${APPDIR}/packages/${p}
	cd ${WRKSRC}/packages/${p} && ${COPYTREE_SHARE} dist \
		${STAGEDIR}${APPDIR}/packages/${p}
.endfor
.for p in shared backend cli thinker
	${MKDIR} ${STAGEDIR}${APPDIR}/packages/${p}
	${INSTALL_DATA} ${WRKSRC}/packages/${p}/package.json \
		${STAGEDIR}${APPDIR}/packages/${p}
.endfor
	cd ${WRKSRC}/packages/cli && ${COPYTREE_SHARE} bin ${STAGEDIR}${APPDIR}/packages/cli
	${FIND} ${STAGEDIR}${APPDIR} -name '*.map' -type f -delete
	${INSTALL_SCRIPT} ${WRKDIR}/thinker-run ${STAGEDIR}${PREFIX}/libexec/thinker-run
	${INSTALL_SCRIPT} ${WRKDIR}/thinker-cli ${STAGEDIR}${PREFIX}/bin/thinker
	${MKDIR} ${STAGEDIR}${ETCDIR} ${STAGEDIR}${PREFIX}/etc/newsyslog.conf.d
	${INSTALL_DATA} ${FILESDIR}/thinker.env.sample ${STAGEDIR}${ETCDIR}
	${INSTALL_DATA} ${FILESDIR}/newsyslog.conf \
		${STAGEDIR}${PREFIX}/etc/newsyslog.conf.d/thinker.conf.sample
	cd ${STAGEDIR}${PREFIX} && ${FIND} libexec/${PORTNAME} \( -type f -o -type l \) \
		| ${SORT} >> ${TMPPLIST}

# Mantenedor: gera o distfile com o cache do npm (rede liberada), para
# amd64 e aarch64, e o deixa em DISTDIR. Depois: make makesum.
make-npm-cache:
	${MAKE} -C ${.CURDIR} DISTFILES="${DISTNAME}${EXTRACT_SUFX}" extract
	${RM} -r ${WRKDIR}/npm-cache ${WRKDIR}/npm-fetch
.for cpu in x64 arm64
	${MKDIR} ${WRKDIR}/npm-fetch
	cd ${WRKSRC} && ${CP} package.json package-lock.json ${WRKDIR}/npm-fetch/ && \
		${TAR} -cf - packages/*/package.json | ${TAR} -xf - -C ${WRKDIR}/npm-fetch
	cd ${WRKDIR}/npm-fetch && ${SETENV} HOME=${WRKDIR} ${LOCALBASE}/bin/npm ci \
		--cache ${WRKDIR}/npm-cache --os freebsd --cpu ${cpu} \
		--ignore-scripts --no-audit --no-fund --loglevel=error
	${RM} -r ${WRKDIR}/npm-fetch
.endfor
	${RM} -r ${WRKDIR}/npm-cache/_logs ${WRKDIR}/npm-cache/_update-notifier-last-checked
	cd ${WRKDIR} && ${TAR} --uid 0 --gid 0 --uname root --gname wheel \
		-cJf ${DISTDIR}/${NPM_CACHE_DIST} npm-cache
	@${ECHO_MSG} "===> ${DISTDIR}/${NPM_CACHE_DIST} pronto; rode make makesum"

.include <bsd.port.mk>
