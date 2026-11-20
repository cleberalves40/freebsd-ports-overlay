PORTNAME=	artex
DISTVERSIONPREFIX=	v
DISTVERSION=	0.6.0
CATEGORIES=	www
MASTER_SITES=	https://github.com/cleberalves40/freebsd-ports-overlay/releases/download/artex-build-cache/:buildcache
DISTFILES=	${NPM_CACHE_DIST}:buildcache \
		${VENDOR_DIST}:buildcache

MAINTAINER=	cleber@bsd.com.br
COMMENT=	Multi-agent LLM pentest platform with embedded UI
WWW=		https://github.com/BaaRai/ARTEX

LICENSE=	AGPLv3
LICENSE_FILE=	${WRKSRC}/LICENSE

ONLY_FOR_ARCHS=	aarch64 amd64
ONLY_FOR_ARCHS_REASON=	Go bootstrap and npm optional native tooling target these ABIs

BUILD_DEPENDS=	npm${NODEJS_SUFFIX}>0:www/npm${NODEJS_SUFFIX}
RUN_DEPENDS=	postgresql18-server>0:databases/postgresql18-server \
		bash>0:shells/bash

USES=		go:1.26,no_targets nodejs:24,build
USE_GITHUB=	yes
GH_ACCOUNT=	BaaRai
GH_PROJECT=	ARTEX
USE_RC_SUBR=	artex artex_db

SUB_FILES=	pkg-message artex-run
SUB_LIST=	APPDIR=${APPDIR} ARTEX_USER=${USERS} DBDIR=/var/db/artex \
		ETCDIR=${PREFIX}/etc/${PORTNAME}

USERS=		artex
GROUPS=		artex
UID_FILES=	${FILESDIR}/UIDs
GID_FILES=	${FILESDIR}/GIDs

# Binario Go estatico: NAO e NO_ARCH
APPDIR=		${PREFIX}/libexec/${PORTNAME}
NPM_CACHE_DIST=	${PORTNAME}-npm-cache-${DISTVERSION}.tar.xz
VENDOR_DIST=	${PORTNAME}-vendor-${DISTVERSION}.tar.xz
NPM_ENV=	HOME=${WRKDIR} npm_config_cache=${WRKDIR}/npm-cache \
		npm_config_offline=true npm_config_audit=false \
		npm_config_fund=false npm_config_update_notifier=false
NPM_CMD=	${SETENV} ${MAKE_ENV} ${NPM_ENV} ${LOCALBASE}/bin/npm

# Extrai caches de build (npm + vendor) no WRKDIR / WRKSRC
post-extract:
	@${ECHO_MSG} "===> Extracting npm cache and Go vendor tree"
	cd ${WRKDIR} && ${TAR} -xf ${DISTDIR}/${NPM_CACHE_DIST}
	cd ${WRKSRC} && ${TAR} -xf ${DISTDIR}/${VENDOR_DIST}

do-build:
	@${ECHO_MSG} "===> Building static UI (NEXT_EXPORT=1, webpack)"
	cd ${WRKSRC}/web && ${NPM_CMD} ci --ignore-scripts --loglevel=error
	cd ${WRKSRC}/web && ${SETENV} ${MAKE_ENV} ${NPM_ENV} \
		NEXT_EXPORT=1 ${LOCALBASE}/bin/npx --yes next build --webpack
	${RM} -r ${WRKSRC}/server/webui/dist
	${MKDIR} ${WRKSRC}/server/webui
	${CP} -a ${WRKSRC}/web/out ${WRKSRC}/server/webui/dist
	@${ECHO_MSG} "===> Building artex (CGO_ENABLED=0, -tags embedui, -mod=vendor)"
	cd ${WRKSRC} && ${SETENV} ${MAKE_ENV} CGO_ENABLED=0 \
		${GO_CMD} build -mod=vendor -tags embedui -trimpath \
		-ldflags "-s -w -X main.version=${DISTVERSION}" \
		-o ${WRKDIR}/artex-bin ./cmd/artex

do-install:
	${MKDIR} ${STAGEDIR}${APPDIR}
	${INSTALL_PROGRAM} ${WRKDIR}/artex-bin ${STAGEDIR}${APPDIR}/artex
	cd ${WRKSRC} && ${COPYTREE_SHARE} skills ${STAGEDIR}${APPDIR}
	cd ${WRKSRC} && ${COPYTREE_SHARE} extension ${STAGEDIR}${APPDIR}
	${INSTALL_DATA} ${WRKSRC}/LICENSE ${STAGEDIR}${APPDIR}
	${INSTALL_SCRIPT} ${WRKDIR}/artex-run ${STAGEDIR}${PREFIX}/libexec/artex-run
	${MKDIR} ${STAGEDIR}${ETCDIR} ${STAGEDIR}${PREFIX}/etc/newsyslog.conf.d
	${INSTALL_DATA} ${FILESDIR}/artex.env.sample ${STAGEDIR}${ETCDIR}
	${INSTALL_DATA} ${FILESDIR}/newsyslog.conf \
		${STAGEDIR}${PREFIX}/etc/newsyslog.conf.d/artex.conf.sample
	cd ${STAGEDIR}${PREFIX} && ${FIND} libexec/${PORTNAME} \( -type f -o -type l \) \
		| ${SORT} >> ${TMPPLIST}

# Mantenedor: gera o distfile do cache npm (rede liberada) para freebsd x64 e arm64.
# Depois: make makesum (com os dois DISTFILES de cache + o GH0).
make-npm-cache:
	${MAKE} -C ${.CURDIR} DISTFILES="${DISTNAME}${EXTRACT_SUFX}" extract
	${RM} -r ${WRKDIR}/npm-cache ${WRKDIR}/npm-fetch
.for cpu in x64 arm64
	${MKDIR} ${WRKDIR}/npm-fetch
	${CP} ${WRKSRC}/web/package.json ${WRKSRC}/web/package-lock.json ${WRKDIR}/npm-fetch/
	cd ${WRKDIR}/npm-fetch && ${SETENV} HOME=${WRKDIR} ${LOCALBASE}/bin/npm ci \
		--cache ${WRKDIR}/npm-cache --os freebsd --cpu ${cpu} \
		--ignore-scripts --no-audit --no-fund --loglevel=error
	${RM} -r ${WRKDIR}/npm-fetch
.endfor
	${RM} -r ${WRKDIR}/npm-cache/_logs ${WRKDIR}/npm-cache/_update-notifier-last-checked
	cd ${WRKDIR} && ${TAR} --uid 0 --gid 0 --uname root --gname wheel \
		-cJf ${DISTDIR}/${NPM_CACHE_DIST} npm-cache
	@${ECHO_MSG} "===> ${DISTDIR}/${NPM_CACHE_DIST} pronto"

# Mantenedor: vendor Go offline (rede liberada). Requer go1.26+.
make-vendor:
	${MAKE} -C ${.CURDIR} DISTFILES="${DISTNAME}${EXTRACT_SUFX}" extract
	cd ${WRKSRC} && ${SETENV} ${MAKE_ENV} CGO_ENABLED=0 ${GO_CMD} mod vendor
	cd ${WRKSRC} && ${TAR} --uid 0 --gid 0 --uname root --gname wheel \
		-cJf ${DISTDIR}/${VENDOR_DIST} vendor
	@${ECHO_MSG} "===> ${DISTDIR}/${VENDOR_DIST} pronto; rode make makesum"

.include <bsd.port.mk>
