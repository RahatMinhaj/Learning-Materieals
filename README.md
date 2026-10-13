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
│       └── EC2-Instance/
│           ├── index.md
│           ├── amazon-ec2-complete-guide.md
│           └── amazon-ec2-complete-guide.html
├── mkdocs.yml
├── requirements.txt
└── .github/workflows/deploy.yml
```

Start with [AWS → EC2 Instance](docs/AWS/EC2-Instance/index.md).

## Local preview

```bash
pip install -r requirements.txt
mkdocs serve
```

Then open http://127.0.0.1:8000/
