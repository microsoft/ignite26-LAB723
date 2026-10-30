<a name="start-building"></a>

<p align="center">
<img src="img/banner-ignite-26.png" alt="Microsoft Ignite 2026" width="100%"/>
</p>

# [Microsoft Ignite 2026](https://ignite.microsoft.com)

## 🔥LAB723: Build, observe, and scale AI-ready apps and workloads with AKS
### Session description

Deploy and operate AI workloads on Azure Kubernetes Service (AKS) without spending the lab building a cluster. In this 75-minute hands-on lab, you deploy a prepared inference service to AKS Automatic, work with the platform's built-in guardrails, tune the workload's resource profile, add GPU acceleration, and watch the service scale under load. A final exercise runs a distributed batch job with Ray on the same cluster.

### 🚀 Getting started

#### In a guided session

If you're following along during a live session:

1. Use the lab environment at your seat. Your AKS Automatic cluster, the lab VM and its tools, and the lab files are already set up, so you don't need your own Azure subscription.
2. Follow the instructions in the lab's instruction pane, starting with Exercise 1, and ask a proctor if you get stuck.
3. Open [`instructions/`](instructions/README.md) to keep a copy of the lab steps for later.

#### On your own

If you're learning at your own pace:

1. Clone this repository, and the lab content repository, [`ai-lab`](https://aka.ms/ignite26-ai-lab), which has the manifests that the exercises apply, the sample app and its container images, the Ray notebook, and the dashboard.
2. Set up an environment like the hosted lab's, as [The lab environment](docs/lab-environment.md) describes. You need an Azure subscription where you can create an AKS Automatic cluster and assign roles, and GPU quota for the optional accelerated profile.
3. Follow the lab steps in [`instructions/`](instructions/README.md), and run the commands from the `labfiles` folder of your `ai-lab` clone.

### 🎯 Learning outcomes

By the end of this session, you will be able to:

- Deploy an AI inference service to AKS Automatic, and make a noncompliant manifest comply with the platform's guardrails.
- Configure and compare CPU and GPU resource profiles for an AI workload, and explain how node autoprovisioning creates the compute that each one needs.
- Use metrics to autoscale the workload under load, relate pod and node scaling to capacity and cost, and identify when a distributed framework such as Ray fits.

### 💻 Technologies used

- Azure Kubernetes Service (AKS) Automatic, with node autoprovisioning and deployment safeguards
- AKS-managed GPU nodes with NVIDIA GPUs
- Microsoft Entra Workload ID
- Azure Monitor managed service for Prometheus, Container insights, and dashboards with Grafana
- Application Insights and OpenTelemetry
- Application routing add-on with managed NGINX ingress
- Kubernetes Horizontal Pod Autoscaler
- Azure Container Registry and Azure Blob Storage
- Ray, KubeRay, and JupyterLab
- ONNX Runtime, with a DistilBERT sentiment model from Hugging Face

### 📚 Continue your learning

Pick your next step based on your learning style:

| Resource | What you'll get |
|----------|-----------------|
| **[Ignite 2026 Next Steps](https://aka.ms/ignite26-next-steps)** | Additional session repos and materials from Microsoft Ignite 2026 |
| **[Microsoft AI Skills Navigator](https://aka.ms/ignite26-asn)** | Discover AI skills, learning paths, and resources for your next step |
| **[Microsoft Learn](https://aka.ms/ignite26-learn)** | Official documentation and guided learning paths on these topics |
| **[Microsoft Foundry Community](https://aka.ms/MicrosoftFoundryDiscord-Ignite26)** | Connect with other learners and experts in our Discord community |
| **[AKS Automatic documentation](https://learn.microsoft.com/azure/aks/intro-aks-automatic)** | Concepts, quickstarts, and how-to guides for the cluster that this lab uses |
| **[Lab content repository](https://aka.ms/ignite26-ai-lab)** | The sample app, manifests, Ray notebook, and dashboard from this lab |

### 🌟 Microsoft Learn MCP Server

The Microsoft Learn MCP Server gives your AI agent direct access to Microsoft's official documentation — grounded, up-to-date answers about the topics in this session.

**GitHub Copilot CLI** — Install with:

```shell
copilot plugin install microsoftdocs/mcp
```

**VS Code** — One-click install:  
[![Install in VS Code](https://img.shields.io/badge/VS_Code-Install_Microsoft_Learn_MCP-0098FF?style=flat-square&logo=visualstudiocode&logoColor=white)](https://vscode.dev/redirect/mcp/install?name=microsoft-learn&config=%7B%22type%22%3A%22http%22%2C%22url%22%3A%22https%3A%2F%2Flearn.microsoft.com%2Fapi%2Fmcp%22%7D)

For more information, visit the [Learn MCP Server repo](https://aka.ms/learnmcp).

### 👥 Content owners

<table>
<tr>
    <td align="center"><a href="http://github.com/bbenz">
        <img src="https://github.com/bbenz.png" width="100px;" alt="Brian Benz"/><br />
        <sub><b>Brian Benz</b></sub></a><br />
            <a href="https://github.com/bbenz" title="talk">📢</a>
    </td>
</tr></table>

### ⚖️ Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft trademarks or logos is subject to and must follow [Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/legal/intellectualproperty/trademarks/usage/general). Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.

Any use of third-party trademarks or logos are subject to those third-party's policies.
