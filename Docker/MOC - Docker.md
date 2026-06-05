---
tags:
  - moc
  - docker
---

# Docker — Map of Content

> Containers = processos Linux com namespaces + cgroups. Docker é a engine que empacota, distribui e orquestra esses processos. Os fundamentos são tecnologia do Kernel Linux; Docker é apenas uma interface amigável sobre eles.

---

## Fundamentos de Containers
- [[Container vs VM]]
- [[Namespaces]]
- [[Cgroups]]
- [[Filesystems e OverlayFS]]
- [[Processo raiz (PID 1)]]
- [[Containers em outros OS]]

## Container Runtime
- [[OCI]]
- [[RunC]]
- [[Como usar o runc]]
- [[Container.d]]
- [[Runtime e Engine]]

## Docker
- [[O que é]]
- [[Arquitetura]]
- [[Imagens]]
- [[Containers]]
- [[Dockerfile]]
- [[Docker Containers]]
- [[Volumes e Persistência]]
- [[Networking]]
- [[Docker Compose]]

## Boas Práticas
- [[Segurança]]
- [[Debugging e Troubleshooting]]
- [[Comandos]]

---

## Conexões com Sistemas Operacionais
Namespaces e Cgroups são recursos do kernel → [[Processos]], [[System Calls]]
OverlayFS = filesystem em camadas → [[Arquivos]]
PID 1 = init process → [[Hierarquia de Processos]], [[O Modelo de Processos]]
Containers vs VMs → [[Máquinas Virtuais Redescobertas]]
Segurança (Seccomp, Capabilities) → [[Hardware de Proteção]], [[Proteção]]

## Conexões com Go
Docker, containerd escritos em Go → [[Goroutines]], [[HTTP (net-http)]]
Docker daemon REST API → [[HTTP (net-http)]]
Container = processo Go pode gerenciar → [[Sistema de Módulos (go mod)]]
