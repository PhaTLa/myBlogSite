# Lâm's DevOps Blog

Welcome to my professional blog where I share insights, best practices, and real-world experiences in DevOps, AWS, Kubernetes, and Infrastructure as Code.

## 🌐 Live Site

Visit the blog at: [https://alexpham15010305.github.io](https://alexpham15010305.github.io) *(Update with your actual domain)*

## 🚀 About

This blog is powered by [Hugo](https://gohugo.io/) using the [LoveIt](https://github.com/dillonzq/LoveIt) theme. It focuses on:

- **DevOps Best Practices** - Real-world strategies and implementations
- **AWS Cloud Architecture** - From basics to advanced patterns
- **Kubernetes & Container Orchestration** - Practical insights and tutorials
- **Infrastructure as Code** - Terraform, Ansible, and automation techniques
- **CI/CD Pipelines** - Modern deployment strategies
- **Monitoring & Observability** - Building reliable systems

## 👨‍💻 Author

**Phạm Tùng Lâm**
- DevOps Engineer & Solution Architect at LG CNS
- Specializing in AWS, Kubernetes, and Infrastructure Automation
- [LinkedIn](https://www.linkedin.com/in/alexpham15010305) | [Email](mailto:phamtunglam.workmail.public@gmail.com)

## 🛠️ Technical Stack

- **Static Site Generator**: Hugo v0.120+
- **Theme**: LoveIt
- **Hosting**: GitHub Pages (or your preferred hosting)
- **CI/CD**: GitHub Actions
- **Comments**: (Optional - configure in hugo.toml)
- **Analytics**: (Optional - configure in hugo.toml)

## 🏗️ Development Setup

### Prerequisites

- [Hugo Extended](https://gohugo.io/installation/) v0.120.0 or later
- [Git](https://git-scm.com/)
- [Node.js](https://nodejs.org/) (optional, for theme development)

### Local Development

1. **Clone the repository**
   ```bash
   git clone https://github.com/alexpham15010305/blog.git
   cd blog
   ```

2. **Initialize theme submodule**
   ```bash
   git submodule update --init --recursive
   ```

3. **Start development server**
   ```bash
   hugo server -D
   ```

4. **Visit your site**
   Open [http://localhost:1313](http://localhost:1313) in your browser

### Creating New Posts

```bash
# Create a new blog post
hugo new posts/my-new-post.md

# Create a new page
hugo new pages/my-new-page.md
```

### Building for Production

```bash
# Build static files
hugo --minify

# The built site will be in the 'public' directory
```

## 📝 Content Guidelines

### Blog Post Structure

```markdown
+++
date = '2025-10-08T10:30:00+07:00'
draft = false
title = 'Your Post Title'
description = "Brief description for SEO and social sharing"
author = 'Phạm Tùng Lâm'
tags = ['AWS', 'DevOps', 'Kubernetes']
categories = ['DevOps']
featured = true
toc = true
+++

# Your Content Here
```

### Recommended Tags

- **Technologies**: AWS, Kubernetes, Docker, Terraform, Ansible
- **Practices**: DevOps, CI/CD, Infrastructure, Monitoring
- **Concepts**: Automation, Security, Scalability, Performance

### Recommended Categories

- DevOps
- AWS
- Kubernetes
- Infrastructure
- Automation
- Monitoring
- Personal

## 🎨 Customization

### Theme Configuration

The main configuration is in `hugo.toml`. Key sections:

- **Site Information**: Title, description, author details
- **Navigation**: Menu items and structure
- **Social Links**: Professional profiles
- **SEO Settings**: Meta tags, Open Graph, Twitter Cards
- **Features**: Search, comments, analytics (optional)

### Custom Styling

To customize the theme:

1. Create `assets/css/custom.css`
2. Add your custom styles
3. The theme will automatically include them

### Adding Images

Place images in the `static/images/` directory. Required images:

- `avatar.png` - Profile photo (400x400px recommended)
- `favicon.ico` - Site favicon
- `og-image.png` - Social media preview (1200x630px)

## 🚀 Deployment

### GitHub Pages

1. **Create GitHub repository** named `username.github.io`
2. **Push your code** to the repository
3. **Configure GitHub Actions** (see `.github/workflows/hugo.yml`)
4. **Enable GitHub Pages** in repository settings

### Manual Deployment

```bash
# Build the site
hugo --minify

# Deploy the 'public' directory to your web server
rsync -avz --delete public/ user@yourserver:/path/to/webroot/
```

## 📊 Performance & SEO

The blog is optimized for:

- **Fast Loading**: Minified CSS/JS, optimized images
- **SEO Friendly**: Proper meta tags, structured data, sitemap
- **Mobile Responsive**: Works great on all devices
- **Accessibility**: Semantic HTML, proper alt tags
- **Social Sharing**: Open Graph and Twitter Card support

## 🤝 Contributing

While this is a personal blog, suggestions and corrections are welcome:

1. **Fork** the repository
2. **Create** a feature branch
3. **Make** your changes
4. **Submit** a pull request

## 📄 License

Content is licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/).
Code is licensed under [MIT License](LICENSE).

## 🙏 Acknowledgments

- [Hugo](https://gohugo.io/) - The world's fastest framework for building websites
- [LoveIt Theme](https://github.com/dillonzq/LoveIt) - A clean, elegant but advanced Hugo theme
- [Font Awesome](https://fontawesome.com/) - Icons used throughout the site

---

**Happy coding and keep learning!** 🚀

*For questions or collaboration opportunities, feel free to reach out on [LinkedIn](https://www.linkedin.com/in/alexpham15010305) or via [email](mailto:phamtunglam.workmail.public@gmail.com).*