graph TB
    subgraph BB[BitBucket - Source Code Repository]
        BB_DEV[develop branch<br/>→ OCP UAT]
        BB_REL[release branch<br/>→ OCP TEST]
        BB_MASTER[master branch<br/>→ OCP PROD]
        BB_WEBHOOK[Webhook<br/>payload: branch, repo,<br/>commit, email]
    end

    subgraph GL[GitLab - CI/CD Platform]
        subgraph GL_REPOS[Pipeline Groups]
            GL_MIRROR[bitbucket-code<br/>Mirror repozitorijumi<br/>po granama]
            GL_MIRROR_PIPE[bitbucket-mirroring<br/>.gitlab-ci.yml<br/>auto clone BB → GL]
            GL_PIPELINE[pipeline-repo<br/>Centralni YAML<br/>OCP vars + templates]
        end
        
        subgraph PIPELINE[CI/CD Pipeline Jobs]
            J1[1. Clone Job<br/>BB → GL]
            J2[2. Initialize Job<br/>branch → env mapping<br/>image naming<br/>artifacts<br/>SAST/XRAY/SECRET vars]
            J3[3. SAST + Secret Detection<br/>GitLab templates]
            J4[4. Build Job<br/>Docker build<br/>+ BB_COMMIT_SHORT_SHA]
            J5[5. X-Ray Scan Job<br/>jf docker scan<br/>report if critical/high]
            J6[6. JFrog Push Job<br/>push image<br/>delete from GL runner]
            J7[7. ArgoCD Sync Job<br/>POST request<br/>sync app per branch]
            J8[8. Pipeline Fail Notify<br/>trigger: on_failure]
        end
    end

    subgraph JFROG[JFrog Artifactory]
        JFROG_IMAGES[Docker Images<br/>Registry]
    end

    subgraph GITOPS[GitOps - DevOps/OCP Projects]
        GITOPS_REPOS[GitLab Repos<br/>per OCP namespace]
        GITOPS_HELM[Helm Charts<br/>per microservice]
        GITOPS_GLOBAL[Global Configuration<br/>Helm Charts:<br/>- Ingress templates<br/>- Network policies<br/>- Global secrets]
    end

    subgraph OCP[OpenShift Container Platform]
        subgraph OCP_TEST[OCP TEST Cluster]
            ARGO_TEST[ArgoCD Operator<br/>OpenShift GitOps]
            TEST_APPS[App-of-Apps<br/>Architecture]
        end
        
        subgraph OCP_PROD[OCP PROD Cluster]
            ARGO_PROD[ArgoCD Operator<br/>OpenShift GitOps]
            PROD_APPS[App-of-Apps<br/>Architecture]
        end
        
        subgraph OCP_UAT[OCP UAT]
            UAT_APPS[Applications]
        end
    end

    %% Flow connections
    BB_WEBHOOK -->|Merge trigger| GL_MIRROR_PIPE
    GL_MIRROR_PIPE -->|Auto clone| GL_MIRROR
    GL_MIRROR --> J1
    GL_PIPELINE -.->|Provides templates| PIPELINE
    
    J1 --> J2
    J2 --> J3
    J3 --> J4
    J4 --> J5
    J5 --> J6
    J6 --> J7
    
    J6 -->|Push image| JFROG_IMAGES
    J7 -->|Sync request| ARGO_TEST
    J7 -->|Sync request| ARGO_PROD
    J7 -->|Sync request| UAT_APPS
    
    J1 -.->|on_failure| J8
    J2 -.->|on_failure| J8
    J3 -.->|on_failure| J8
    J4 -.->|on_failure| J8
    J5 -.->|on_failure| J8
    J6 -.->|on_failure| J8
    J7 -.->|on_failure| J8
    
    ARGO_TEST -->|Watches| GITOPS_REPOS
    ARGO_PROD -->|Watches| GITOPS_REPOS
    GITOPS_REPOS --> GITOPS_HELM
    GITOPS_REPOS --> GITOPS_GLOBAL
    
    ARGO_TEST -->|Pulls images| JFROG_IMAGES
    ARGO_PROD -->|Pulls images| JFROG_IMAGES
    
    GITOPS_HELM --> TEST_APPS
    GITOPS_HELM --> PROD_APPS
    GITOPS_GLOBAL --> TEST_APPS
    GITOPS_GLOBAL --> PROD_APPS

    %% Styling
    classDef bbStyle fill:#0052CC,color:#fff,stroke:#333
    classDef glStyle fill:#FC6D26,color:#fff,stroke:#333
    classDef jfrogStyle fill:#41BF47,color:#fff,stroke:#333
    classDef ocpStyle fill:#EE0000,color:#fff,stroke:#333
    classDef argoStyle fill:#FF6E42,color:#fff,stroke:#333
    classDef jobStyle fill:#63666A,color:#fff,stroke:#333
    
    class BB,BB_DEV,BB_REL,BB_MASTER,BB_WEBHOOK bbStyle
    class GL_MIRROR,GL_MIRROR_PIPE,GL_PIPELINE glStyle
    class J1,J2,J3,J4,J5,J6,J7,J8 jobStyle
    class JFROG_IMAGES jfrogStyle
    class ARGO_TEST,ARGO_PROD argoStyle
    class TEST_APPS,PROD_APPS,UAT_APPS ocpStyle