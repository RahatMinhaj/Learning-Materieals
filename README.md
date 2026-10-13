# Learning Materials

A navigable collection of practical study notes.

**Site:** https://rahatminhaj.github.io/Learning-Materieals/

## Subjects

- [AWS](docs/AWS/index.md)

## Repository layout

```text
Learning-Materieals/
├── docs/
│   ├── index.md
│   └── AWS/
│       ├── index.md
│       ├── Getting-Started/
│       │   ├── index.md
│       │   ├── aws-from-zero-to-deployed.md
│       │   └── aws-from-zero-to-deployed.html
│       ├── EC2-Instance/
│       │   ├── index.md
│       │   ├── amazon-ec2-complete-guide.md
│       │   └── amazon-ec2-complete-guide.html
│       └── IAM/
│           ├── index.md
│           ├── amazon-iam-complete-guide.md
│           └── amazon-iam-complete-guide.html
├── mkdocs.yml
├── requirements.txt
└── .github/workflows/deploy.yml
```

Start with [AWS → Getting Started](docs/AWS/Getting-Started/index.md) if you are new to AWS.

## Local preview

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open http://127.0.0.1:8000/
