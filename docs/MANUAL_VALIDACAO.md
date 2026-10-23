# Manual de validação — php2shell PoC

Guia prático para (1) reproduzir o comportamento vulnerável, (2) confirmar que a
versão/aplicação corrigida responde de forma diferente, e (3) interpretar os
resultados. Cobre as duas CVEs do `session.upload_progress`: CVE-2026-109618 e
CVE-2026-109619.

> **Antes de começar:** o scanner padrão é não destrutivo. A única sonda que pode
> derrubar workers do PHP fica atrás da flag `--active`. **Nunca use `--active`
> contra produção.** Rode contra staging sempre que possível.

---

## 1. Pré-requisitos

- Docker e Docker Compose (só para o laboratório de referência).
- Python 3.8+ (os scripts usam apenas a biblioteca padrão, sem dependências).
- Acesso de rede ao alvo que você quer validar.

Nenhum script faz chamada externa: todo destino vem do argumento `--target` que
você passa. Os defaults apontam para `127.0.0.1`.

---

## 2. Estratégia de validação

A ideia é comparar **duas respostas**:

1. **Alvo vulnerável** — mostra como é a falha "passando".
2. **Alvo corrigido** — deve responder de forma diferente (falha não passa mais).

Você tem duas formas de obter o alvo vulnerável de referência:

- Subir o laboratório local (seção 3), que já traz versões vulneráveis **e** a
  versão corrigida lado a lado.
- Usar uma cópia da sua aplicação antes e depois da mitigação.

---

## 3. Subir o laboratório de referência (opcional, recomendado)

O lab sobe várias versões de PHP em portas diferentes. Ele baixa apenas imagens
oficiais (`php:*`, `nginx:alpine`) do Docker Hub.

```bash
cd lab
./setup.sh          # cria o docker-compose e sobe os containers
```

### Mapa de portas (localhost)

| Porta  | Versão PHP    | Estado          |
|--------|---------------|-----------------|
| 18056  | 5.6.40        | vulnerável (EOL)|
| 18074  | 7.4.33        | vulnerável (EOL)|
| 18080  | 8.0.30        | vulnerável (EOL)|
| 18081  | 8.1.31        | vulnerável (EOL)|
| 18082  | 8.2.34        | **vulnerável**  |
| 18083  | 8.3.35        | **vulnerável**  |
| 18084  | 8.4.26        | **vulnerável**  |
| 18085  | 8.2.35        | **corrigido**   |
| 18091  | 8.2.34-apache | vulnerável (mod_php) |

Use **18082** como alvo vulnerável e **18085** como alvo corrigido. São a mesma
linha 8.2, diferindo só no patch — o par ideal de comparação.

Para derrubar o lab quando terminar:

```bash
cd lab
docker compose down
```

---

## 4. Validar o erro (comportamento vulnerável)

### 4.1 Scanner — detecção não destrutiva

```bash
python3 scanner/check.py http://127.0.0.1:18082/
```

Saída esperada em um alvo vulnerável:

```
verdict: VULNERABLE   (confidence: HIGH)
```

O scanner envia apenas sondas de detecção: fingerprint de versão, comportamento
do `upload_progress`, fixação de sessão e um marcador de escrita de sessão.

### 4.2 Exploit — prova de execução (chain A)

Para provar RCE de ponta a ponta contra o alvo vulnerável. O comando padrão é
inofensivo (`id`):

```bash
python3 exploit/chain_a_session_lfi.py --target http://127.0.0.1:18082/ --command id
```

Sinais de sucesso (alvo vulnerável):

- Imprime `[+] RCE CONFIRMED: <saída do id>`.
- **Exit code 0.**

Confira o código de saída:

```bash
echo $?        # 0 = RCE confirmado; 1 = não confirmado
```

---

## 5. Validar que o fix responde diferente

Repita **exatamente os mesmos comandos**, trocando só o alvo para a versão
corrigida (ou para a sua aplicação já mitigada).

### 5.1 Scanner contra o alvo corrigido

```bash
python3 scanner/check.py http://127.0.0.1:18085/
```

Saída esperada em um alvo corrigido:

```
verdict: PATCHED   (confidence: HIGH | MEDIUM)
```

O scanner decide pela versão do PHP. São consideradas corrigidas:

| Linha | Primeira versão corrigida |
|-------|---------------------------|
| 8.2   | 8.2.35                    |
| 8.3   | 8.3.36                    |
| 8.4   | 8.4.27                    |
| 8.5   | 8.5.12                    |

### 5.2 Exploit contra o alvo corrigido

```bash
python3 exploit/chain_a_session_lfi.py --target http://127.0.0.1:18085/ --command id
echo $?
```

Sinais de que o fix pegou (resposta diferente):

- Imprime `[-] RCE not confirmed — LFI path/session path or timing is wrong`.
- **Exit code 1.**

### 5.3 Contra a sua própria aplicação

Troque o alvo pela URL do seu endpoint. Use **staging**, sem `--active`:

```bash
python3 scanner/check.py https://staging.seu-app.com/endpoint
```

Mesmo o modo padrão envia POSTs reais que criam sessões. Isso é baixo risco, mas
aponte para staging primeiro.

---

## 6. Tabela de interpretação

### Scanner (`scanner/check.py`)

| Veredito             | Significado                                               |
|----------------------|-----------------------------------------------------------|
| `VULNERABLE`         | Versão vulnerável e/ou `upload_progress` alcançável.      |
| `LIKELY_VULNERABLE`  | `upload_progress` ativo, mas versão não identificada.     |
| `PATCHED`            | Versão igual ou acima do patch. **Objetivo da validação.**|
| `UNKNOWN`            | Sem sinal suficiente (ex.: banner do PHP escondido).      |
| `ERROR`              | Falha de conexão com o alvo.                              |

Se a versão não aparece no banner, o veredito pode cair em `UNKNOWN`. Nesse caso
confirme a versão do PHP por outro meio e/ou rode o exploit da seção 5.2.

### Exploit (`exploit/chain_a_session_lfi.py`)

| Exit code | Saída                      | Conclusão                    |
|-----------|----------------------------|------------------------------|
| `0`       | `[+] RCE CONFIRMED: ...`   | Vulnerável — falha passa.     |
| `1`       | `[-] RCE not confirmed`    | Mitigado — falha não passa.   |

---

## 7. Checklist de validação do seu fix

1. [ ] Subi o lab e vi `VULNERABLE` na porta 18082 (referência do erro).
2. [ ] Vi `[+] RCE CONFIRMED` com o exploit na porta 18082 (exit 0).
3. [ ] Vi `PATCHED` na porta 18085 (referência do fix).
4. [ ] Vi `[-] RCE not confirmed` na porta 18085 (exit 1).
5. [ ] Rodei o scanner contra a minha aplicação mitigada e obtive `PATCHED`.
6. [ ] Rodei o exploit contra a minha aplicação e obtive exit code 1.
7. [ ] Em nenhum momento usei `--active` contra produção.

Se os passos 5 e 6 baterem com os passos 3 e 4, o fix está respondendo diferente
do comportamento vulnerável — a mitigação passou.

---

## 8. Segurança e escopo

- **Sem chamadas externas:** nenhum IP/domínio hardcoded, sem telemetria.
- **Payloads padrão inofensivos:** os exploits rodam `id`/`uname -a` e limpam os
  artefatos (`/tmp/.rce`) ao final.
- **Flag destrutiva:** só `scanner/check.py --active` dispara a sonda de OOM, que
  pode derrubar workers. Não use em produção.
- Use apenas em sistemas que você tem autorização para testar.
