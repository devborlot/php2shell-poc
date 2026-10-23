<p align="center">
  <img src="docs/cover.png" alt="php2shell" width="700">
</p>

<h1 align="center">php2shell</h1>
<p align="center">
  <strong>CVE-2026-109618 & CVE-2026-109619 — RCE pré-autenticação no PHP: kit de exploração e detecção</strong>
</p>
<p align="center">
  <a href="https://github.com/ikow/php2shell-poc/stargazers"><img src="https://img.shields.io/github/stars/ikow/php2shell-poc?style=flat-square" alt="Stars"></a>
  <img src="https://img.shields.io/badge/CVSS-9.2%20Critical-red?style=flat-square" alt="CVSS 9.2">
  <img src="https://img.shields.io/badge/PHP-5.4%E2%80%938.5-orange?style=flat-square" alt="Versões afetadas">
  <img src="https://img.shields.io/badge/Fixed-8.2.35%20%7C%208.3.36%20%7C%208.4.27%20%7C%208.5.12-green?style=flat-square" alt="Versões corrigidas">
  <img src="https://img.shields.io/badge/Exploits-3%20public%20%7C%202%20paid-blue?style=flat-square" alt="Exploits">
</p>

<p align="center">
  <a href="#exploits">Exploits</a> · <a href="#regras-de-deteccao">Detecção</a> · <a href="#inicio-rapido">Lab</a> · <a href="docs/MANUAL_VALIDACAO.md">Manual de validação</a> · <a href="https://qideng.substack.com/p/php2shell-dissecting-phps-critical">Writeup</a>
</p>

<p align="center">
  <sub>🇧🇷 Português · <a href="README.en.md">🇬🇧 English (original)</a></sub>
</p>

---

Cinco cadeias independentes de RCE que exploram o `session.upload_progress` do PHP e o bailout por OOM do motor Zend. **14 anos de código vulnerável** — do PHP 5.4.0 (2012) ao 8.5.11. Pré-autenticação. Algumas cadeias não exigem nenhuma vulnerabilidade na aplicação.

Afeta **78% de todos os sites** — incluindo WordPress (43%), Joomla, Drupal, Laravel, Magento e toda instalação PHP padrão.

**Autor:** [Qi Deng](https://github.com/ikow) &nbsp;|&nbsp; **Writeup:** [Substack](https://qideng.substack.com/p/php2shell-dissecting-phps-critical) &nbsp;|&nbsp; **Data:** 2026-10-21

> **Fork em português.** Este repositório é um fork do trabalho original de [Qi Deng (@ikow)](https://github.com/ikow/php2shell-poc), com o README traduzido e um [manual de validação](docs/MANUAL_VALIDACAO.md) adicionado. Toda a pesquisa e autoria original são de Qi Deng.

## Detalhes das CVEs

| CVE | GHSA | CVSS | Descrição |
|-----|------|------|-----------|
| CVE-2026-109618 | [GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) | 9.2 | OOM → `longjmp()` deixa o estado do motor Zend inválido → RCE |
| CVE-2026-109619 | [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) | 9.2 | Estado global obsoleto entre requisições do worker FPM → UAF → RCE |

**Patch:** [Commit 3584a98](https://github.com/php/php-src/commit/3584a98bda) — 300 arquivos alterados no motor Zend.

## Cadeias de ataque

| Cadeia | Vetor | Pré-auth | Condições | SAPI |
|--------|-------|:--------:|-----------|------|
| **A** | Injeção de sessão + LFI | Sim | App tem qualquer sink de LFI | Todos |
| **B** | UAF em var_destroy → sobrescrita de handler | Não | App chama `unserialize()` | Todos |
| **C** | Mismatch de handler → `__destruct` | Sim | Dois diretórios com serialize_handler diferente | FPM |
| **D** | Corrupção por OOM → `EG()` obsoleto | Sim | **Nenhuma** — qualquer endpoint | mod_php |
| **E** | OOM de sessão FPM → `BG()` obsoleto | Sim | Produtor: qualquer endpoint com `session_start()`; Consumidor: **qualquer** endpoint | FPM |

A cadeia D é a mais crítica: **RCE pré-autenticação sem nenhuma dependência** contra qualquer instalação Apache mod_php.

A cadeia E é quase tão abrangente no FPM: o produtor precisa de qualquer endpoint que chame `session_start()` (praticamente universal — WordPress, Laravel, Joomla e Drupal todos chamam), e o **consumidor pode ser qualquer endpoint PHP** — o handler `rfc1867` do upload_progress roda durante a inicialização do SAPI, antes do código da aplicação. Verificado: `nosession.php` (sem `session_start()` no código) provoca crash 502 ao consumir o estado obsoleto de `BG(unserialize)`.

## Exploits

Três exploits de RCE (cadeias A, B, C) estão incluídos em [`exploit/`](exploit/):

| Arquivo | Cadeia | O que faz |
|---------|--------|-----------|
| `chain_a_session_lfi.py` | A | Injeta código PHP pelo nome do arquivo de upload → LFI inclui o arquivo de sessão → RCE |
| `chain_b_uaf_rce.php` | B | UAF em var_destroy → descobre `zif_system` via EG() → sobrescreve o handler de `var_dump` via `/proc/self/mem` → RCE |
| `chain_c_mismatch_rce.py` | C | Polui o `PS(serializer)` do worker FPM → injeta objeto serializado pelo nome do arquivo → `__destruct` dispara na falha de decode → RCE |

Os exploits das cadeias D e E estão disponíveis para [assinantes pagos](https://qideng.substack.com/p/php2shell-dissecting-phps-critical).

```bash
# Cadeia A: injeção de sessão + LFI → RCE
python3 exploit/chain_a_session_lfi.py --host ALVO --port 80 --command 'id'

# Cadeia B: UAF em var_destroy → RCE (rodar no alvo ou via webshell enviada)
php exploit/chain_b_uaf_rce.php

# Cadeia C: mismatch de handler → __destruct → RCE
python3 exploit/chain_c_mismatch_rce.py --host ALVO --port 80 --command 'id'
```

## Regras de detecção

Regras prontas de IDS / WAF / SIEM em [`detection/`](detection/):

| Arquivo | Formato | Assinaturas |
|---------|---------|-------------|
| `php2shell.rules` | Suricata / Snort | 13 regras — requisição, resposta e reconhecimento |
| `php2shell.yaml` | Sigma | 5 regras — logs web, criação de processo, eventos de arquivo |
| `modsecurity.conf` | ModSecurity v2/v3 | 7 regras — plug-and-play para Apache/Nginx |
| `php2shell.lua` | OpenResty Lua | Módulo leve para proxy reverso |

**Mitigação mais rápida:** bloquear `PHP_SESSION_UPLOAD_PROGRESS` em corpos de POST multipart. Uso legítimo praticamente nulo e cobre o vetor de ataque principal das duas CVEs.

## Início rápido

```bash
git clone https://github.com/ikow/php2shell-poc.git && cd php2shell-poc

# Subir o lab (PHP-FPM + nginx + Apache mod_php)
cd lab && docker compose up -d

# Verificar cada cadeia
python3 ../poc/verify_session_injection.py --target http://127.0.0.1:18080
python3 ../poc/verify_handler_mismatch.py --target http://127.0.0.1:18080
python3 ../poc/verify_oom_trigger.py --host 127.0.0.1 --port 18081
python3 ../poc/verify_fpm_stale_state.py --target http://127.0.0.1:18080

# Rodar um exploit
python3 ../exploit/chain_a_session_lfi.py --host 127.0.0.1 --port 18080 --command 'id'
```

> 📘 Para um passo a passo de como reproduzir a falha e confirmar que a versão/aplicação corrigida responde diferente, veja o **[Manual de validação](docs/MANUAL_VALIDACAO.md)**.

## Estrutura do repositório

```
php2shell-poc/
├── exploit/
│   ├── chain_a_session_lfi.py        # RCE: injeção de sessão + LFI
│   ├── chain_b_uaf_rce.php           # RCE: UAF em var_destroy → sobrescrita de handler
│   └── chain_c_mismatch_rce.py       # RCE: mismatch de handler → __destruct
├── detection/
│   ├── php2shell.rules               # Regras de IDS Suricata / Snort
│   ├── php2shell.yaml                # Regras de detecção SIEM (Sigma)
│   ├── modsecurity.conf              # Regras de WAF ModSecurity
│   └── php2shell.lua                 # Módulo WAF OpenResty / Nginx Lua
├── poc/
│   ├── verify_session_injection.py   # Verifica a cadeia A
│   ├── verify_uaf_asan.php           # Verifica a cadeia B (ASAN)
│   ├── verify_handler_mismatch.py    # Verifica a cadeia C
│   ├── verify_oom_trigger.py         # Verifica a cadeia D
│   └── verify_fpm_stale_state.py     # Verifica a cadeia E
├── lab/
│   ├── docker-compose.yml            # PHP-FPM + nginx + Apache mod_php
│   ├── setup.sh                      # Setup do lab em um comando
│   └── php.ini / php-apache.ini      # Configs PHP vulneráveis
└── docs/
    ├── ANALYSIS.md                   # Análise técnica completa
    ├── MANUAL_VALIDACAO.md           # Manual de validação (PT-BR)
    └── substack_article.md           # Fonte do writeup
```

## Mitigação

```ini
; Imediato — reduz a superfície de ataque
session.upload_progress.enabled = Off
session.use_only_cookies = On
```

> **A única correção completa é atualizar o PHP** para 8.2.35 / 8.3.36 / 8.4.27 / 8.5.12 ou superior. A corrupção por OOM (cadeia D) está no núcleo do motor Zend — nenhuma configuração de INI resolve.

## Impacto downstream

A Wiz demonstrou RCE pré-autenticação em imagens de container padrão:

| Plataforma | Vulnerável | Observações |
|------------|:----------:|-------------|
| WordPress | Sim | O core não usa sessões PHP, mas o WooCommerce e muitos plugins usam |
| Joomla | Sim | Usa sessões PHP nativas por padrão |
| Drupal | Sim | Usa sessões PHP nativas por padrão |
| Magento | Sim | Vulnerável mesmo em modo de manutenção |
| Laravel | Sim | `PendingBroadcast` é um gadget `__destruct` conhecido |

## Referências

- [PHP Security Advisory](https://www.php.net/archive/2026.php)
- [GHSA-w8hw-w9fx-j79w](https://github.com/php/php-src/security/advisories/GHSA-w8hw-w9fx-j79w) — CVE-2026-109618
- [GHSA-6wmf-4rv5-p3rp](https://github.com/php/php-src/security/advisories/GHSA-6wmf-4rv5-p3rp) — CVE-2026-109619
- [Commit do patch 3584a98](https://github.com/php/php-src/commit/3584a98bda) — 300 arquivos, motor Zend + extensões
- [Cobertura do WAF da Cloudflare](https://blog.cloudflare.com/php-vulnerabilities-waf-protection/)
- [Análise da Sansec](https://sansec.io/research/php2shell) — descoberta do vazamento de heap via PHPSESSID
- [Impacto downstream pela Wiz](https://www.wiz.io/blog/php2shell-php-rce)
- [Detecção da Corelight](https://github.com/corelight/php2shell-detect) — regras comunitárias Zeek + Suricata

## Aviso legal

Este repositório destina-se **exclusivamente a testes de segurança autorizados e fins educacionais**. O código de exploração das cadeias D e E (RCE sem dependências) está disponível para [assinantes pagos](https://qideng.substack.com/p/php2shell-dissecting-phps-critical) do writeup.

## Licença

MIT

---

<p align="center">
  <sub>
    <b>Palavras-chave:</b> php2shell, CVE-2026-109618, CVE-2026-109619, PHP RCE, PHP exploit, session.upload_progress,
    pre-auth RCE, PHP vulnerability, PHP session injection, PHP deserialization, PHP OOM, PHP memory corruption,
    use-after-free, GHSA-w8hw-w9fx-j79w, GHSA-6wmf-4rv5-p3rp, WordPress RCE, Joomla RCE, Drupal RCE,
    PHP security, WAF rules, Suricata rules, ModSecurity rules, IDS signatures, PHP 8.2.35, PHP 8.4.27,
    session upload progress, PHP session fixation, rfc1867, longjmp, zend_bailout,
    Snort rules, Sigma rules, detection rules, PHP patch, PHP critical vulnerability
  </sub>
</p>
