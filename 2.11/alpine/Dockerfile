FROM alpine:3.24

RUN apk add --no-cache \
	ca-certificates \
	curl \
	libcap \
	mailcap

RUN set -eux; \
	mkdir -p \
		/config/caddy \
		/data/caddy \
		/etc/caddy \
		/usr/share/caddy \
	; \
	chmod 1777 /config/caddy /data/caddy; \
	wget -O /etc/caddy/Caddyfile "https://github.com/caddyserver/dist/raw/33ae08ff08d168572df2956ed14fbc4949880d94/config/Caddyfile"; \
	wget -O /usr/share/caddy/index.html "https://github.com/caddyserver/dist/raw/33ae08ff08d168572df2956ed14fbc4949880d94/welcome/index.html"

# https://github.com/caddyserver/caddy/releases
ENV CADDY_VERSION=v2.11.9

RUN set -eux; \
	apkArch="$(apk --print-arch)"; \
	case "$apkArch" in \
		x86_64)  binArch='amd64'; checksum='0e85544340290a16ce4bfb76fe7d65e9de1bf3a6e67f5ccfff30b30d27ea4c73e078077b7bf3e58ae60b6b12e44327705287059c775bca5782e8ee85ea7edd75' ;; \
		armhf)   binArch='armv6'; checksum='bfd9b415660caccaf5efbba2a8828eb9e61e31609ee6a62d532e70ef1f419e2ba420fdfd1e16c13e4896c33557813f87fddc849de57400eac34c4635f413bc0c' ;; \
		armv7)   binArch='armv7'; checksum='ea042518bfb5eec5070a3119430263ae920b2d1e878c733a80d3d0ced1168afe5816362f295de46915d651a67f31f4d8a67ac67326f507d90cda71274b8ee689' ;; \
		aarch64) binArch='arm64'; checksum='9e5e76d7888542924de2c25353ae87c5f77e2a22c5b1198633cae13688922282e05383d142599205c88d8d80eee5e967d291214d7abce4835329a191b20dd6e0' ;; \
		ppc64el|ppc64le) binArch='ppc64le'; checksum='9b95cfa058e9651c0738179c6c18d0a93eb9c491e5c9ed3d95f30b23e1c9324e739bbf91a3ac0ebf91aaff89e25de1cff22f127484be937c0089b079266cd37f' ;; \
		riscv64) binArch='riscv64'; checksum='52c9750b091a297766f2faaa01955eb894b614f4b94a9a418632d67f65f942e5363681869de0a6434d87482ac611f5c228282c23c6a90349085bb17aa3eff686' ;; \
		s390x)   binArch='s390x'; checksum='213186130eeab27bc8fff06f4a6cffb253b36f719166c6cb12c539b4432186d6f7938847547a4e5cf47b1525424fdc74813950f08b928f788b7402127eb30e6b' ;; \
		*) echo >&2 "error: unsupported architecture ($apkArch)"; exit 1 ;;\
	esac; \
	wget -O /tmp/caddy.tar.gz "https://github.com/caddyserver/caddy/releases/download/v2.11.9/caddy_2.11.9_linux_${binArch}.tar.gz"; \
	echo "$checksum  /tmp/caddy.tar.gz" | sha512sum -c; \
	tar x -z -f /tmp/caddy.tar.gz -C /usr/bin caddy; \
	rm -f /tmp/caddy.tar.gz; \
	setcap cap_net_bind_service=+p /usr/bin/caddy; \
	chmod +x /usr/bin/caddy; \
	caddy version

# See https://caddyserver.com/docs/conventions#file-locations for details
ENV XDG_CONFIG_HOME=/config
ENV XDG_DATA_HOME=/data

LABEL org.opencontainers.image.version=v2.11.9
LABEL org.opencontainers.image.title=Caddy
LABEL org.opencontainers.image.description="a powerful, enterprise-ready, open source web server with automatic HTTPS written in Go"
LABEL org.opencontainers.image.url=https://caddyserver.com
LABEL org.opencontainers.image.documentation=https://caddyserver.com/docs
LABEL org.opencontainers.image.vendor="Light Code Labs"
LABEL org.opencontainers.image.licenses=Apache-2.0
LABEL org.opencontainers.image.source="https://github.com/caddyserver/caddy-docker"

EXPOSE 80
EXPOSE 443
EXPOSE 443/udp
EXPOSE 2019

WORKDIR /srv

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]

# Cloady HTTPS VLESS/WebSocket proxy. Supply SB_UUID, SB_SUB_TOKEN and SB_HOST at runtime.
COPY --from=ghcr.io/sagernet/sing-box:1.14.4 /usr/local/bin/sing-box /usr/local/bin/sing-box
RUN apk add --no-cache python3
RUN <<'BUILD_PROXY'
cat > /usr/local/bin/start-proxy.py <<'PYTHON_PROXY'
import base64, json, os, pathlib, re, signal, subprocess, sys, time, uuid

# Credentials belong in Cloady environment variables, never in this public repo.
user = str(uuid.UUID(os.environ['SB_UUID']))
token = os.environ['SB_SUB_TOKEN']
host = os.environ['SB_HOST']
if not re.fullmatch(r'[a-f0-9]{64}', token):
    raise ValueError('SB_SUB_TOKEN must contain 64 lowercase hex characters')
if not re.fullmatch(r'[a-zA-Z0-9.-]+', host) or '.' not in host:
    raise ValueError('SB_HOST must be a public DNS hostname')
ws = '/connect'
state = pathlib.Path('/run/sing-box')
state.mkdir(mode=0o700, parents=True, exist_ok=True)
os.umask(0o077)
private = ['0.0.0.0/8','10.0.0.0/8','100.64.0.0/10','127.0.0.0/8','169.254.0.0/16','172.16.0.0/12','192.168.0.0/16','198.18.0.0/15','224.0.0.0/4','240.0.0.0/4','::/128','::1/128','::ffff:0:0/96','fc00::/7','fe80::/10','ff00::/8']
server = {'log':{'level':'warn'},'dns':{'servers':[{'type':'local','tag':'system'}]},'inbounds':[{'type':'vless','tag':'vless-in','listen':'127.0.0.1','listen_port':8080,'users':[{'name':'Peter','uuid':user}],'transport':{'type':'ws','path':ws}}],'outbounds':[{'type':'direct','tag':'direct'}],'route':{'default_domain_resolver':'system','rules':[{'action':'resolve'},{'ip_cidr':private,'action':'reject'}],'final':'direct'}}
proxy = {'type':'vless','tag':'proxy','server':host,'server_port':443,'uuid':user,'tls':{'enabled':True,'server_name':host},'transport':{'type':'ws','path':ws,'headers':{'Host':host}}}
client = {'log':{'level':'warn'},'dns':{'servers':[{'type':'local','tag':'local'},{'type':'https','tag':'remote','server':'1.1.1.1','detour':'proxy'}],'final':'remote'},'inbounds':[{'type':'tun','tag':'tun-in','address':['172.19.0.1/30'],'auto_route':True,'strict_route':True,'stack':'mixed'},{'type':'mixed','tag':'mixed-in','listen':'127.0.0.1','listen_port':7890}],'outbounds':[proxy,{'type':'direct','tag':'direct'}],'route':{'auto_detect_interface':True,'default_domain_resolver':'local','rules':[{'action':'sniff'},{'protocol':'dns','action':'hijack-dns'},{'ip_is_private':True,'outbound':'direct'}],'final':'proxy'}}
node = f'vless://{user}@{host}:443?encryption=none&security=tls&sni={host}&type=ws&host={host}&path=%2Fconnect#Peter-Cloady-US' + chr(10)
clash = {'mixed-port':7890,'allow-lan':False,'mode':'rule','log-level':'warning','tun':{'enable':True,'stack':'mixed','auto-route':True,'auto-detect-interface':True,'dns-hijack':['any:53']},'dns':{'enable':True,'nameserver':['https://1.1.1.1/dns-query']},'proxies':[{'name':'Peter-Cloady-US','type':'vless','server':host,'port':443,'uuid':user,'tls':True,'servername':host,'network':'ws','udp':True,'ws-opts':{'path':ws,'headers':{'Host':host}}}],'proxy-groups':[{'name':'PROXY','type':'select','proxies':['Peter-Cloady-US']}],'rules':['MATCH,PROXY']}

def response(path, body, content_type):
    return {'match':[{'path':[path]}],'handle':[{'handler':'static_response','headers':{'Content-Type':[content_type],'Cache-Control':['no-store'],'Referrer-Policy':['no-referrer']},'body':body}],'terminal':True}

prefix = '/sub/' + token
routes = [response(prefix, base64.b64encode(node.encode()).decode(), 'text/plain; charset=utf-8'),response(prefix+'/sing-box.json',json.dumps(client),'application/json'),response(prefix+'/clash.yaml',json.dumps(clash),'application/yaml'),response(prefix+'/node',node,'text/plain; charset=utf-8'),{'match':[{'path':[ws]}],'handle':[{'handler':'reverse_proxy','upstreams':[{'dial':'127.0.0.1:8080'}]}],'terminal':True},response('/healthz','ok','text/plain'),{'handle':[{'handler':'static_response','status_code':404,'body':'Not found'}]}]
caddy = {'admin':{'disabled':True},'apps':{'http':{'servers':{'proxy':{'listen':[':3000'],'routes':routes}}}}}
for name, data in [('server.json',server),('caddy.json',caddy)]:
    (state/name).write_text(json.dumps(data))
subprocess.run(['sing-box','check','-c',str(state/'server.json')],check=True)
children = []
def stop(signum=None, frame=None):
    for child in children:
        if child.poll() is None:
            child.terminate()
for sig in (signal.SIGTERM,signal.SIGINT):
    signal.signal(sig,stop)
children.append(subprocess.Popen(['sing-box','run','-c',str(state/'server.json')]))
children.append(subprocess.Popen(['caddy','run','--config',str(state/'caddy.json')]))
try:
    while all(child.poll() is None for child in children):
        time.sleep(1)
finally:
    stop()
    for child in children:
        try:
            child.wait(timeout=10)
        except subprocess.TimeoutExpired:
            child.kill()
    sys.exit(1)
PYTHON_PROXY
python3 -m py_compile /usr/local/bin/start-proxy.py
BUILD_PROXY
EXPOSE 3000
CMD ["python3", "/usr/local/bin/start-proxy.py"]
