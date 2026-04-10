# Analyze Project Pattern

This pattern demonstrates how to analyze a go-kratos project structure.

## Purpose

Analyze an existing go-kratos project to understand:
- Project structure and organization
- API definitions and endpoints
- Business logic implementation
- Data access patterns
- Dependencies and wiring

## When to Use

- Onboarding to an existing project
- Code review preparation
- Refactoring planning
- Documentation generation

## Implementation

### Step 1: Gather Project Information

```bash
#!/bin/bash
# analyze-project.sh

echo "=== Project Analysis ==="
echo ""

echo "1. Project Structure:"
find . -type f -name "*.go" | head -20

echo ""
echo "2. Proto Files:"
find . -name "*.proto" -type f

echo ""
echo "3. Dependencies:"
grep "go-kratos" go.mod

echo ""
echo "4. Services:"
grep -r "type.*Service struct" internal/service/

echo ""
echo "5. API Endpoints:"
grep -r "rpc " api/
```

### Step 2: Analyze Project Structure

```go
// analyze.go
package main

import (
    "fmt"
    "go/ast"
    "go/parser"
    "go/token"
    "os"
    "path/filepath"
    "strings"
)

type ProjectAnalysis struct {
    Name        string
    Services    []ServiceInfo
    APIs        []APIInfo
    Models      []ModelInfo
    Middlewares []string
}

type ServiceInfo struct {
    Name    string
    Methods []string
}

type APIInfo struct {
    Service string
    Method  string
    Input   string
    Output  string
}

type ModelInfo struct {
    Name   string
    Fields []string
}

func AnalyzeProject(projectPath string) (*ProjectAnalysis, error) {
    analysis := &ProjectAnalysis{
        Name: filepath.Base(projectPath),
    }
    
    // Walk through project files
    err := filepath.Walk(projectPath, func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        
        // Analyze Go files
        if strings.HasSuffix(path, ".go") && !strings.Contains(path, "vendor") {
            if err := analyzeGoFile(path, analysis); err != nil {
                return err
            }
        }
        
        // Analyze proto files
        if strings.HasSuffix(path, ".proto") {
            if err := analyzeProtoFile(path, analysis); err != nil {
                return err
            }
        }
        
        return nil
    })
    
    return analysis, err
}

func analyzeGoFile(path string, analysis *ProjectAnalysis) error {
    fset := token.NewFileSet()
    node, err := parser.ParseFile(fset, path, nil, parser.ParseComments)
    if err != nil {
        return err
    }
    
    // Find service structs
    ast.Inspect(node, func(n ast.Node) bool {
        if ts, ok := n.(*ast.TypeSpec); ok {
            if _, ok := ts.Type.(*ast.StructType); ok {
                name := ts.Name.Name
                if strings.HasSuffix(name, "Service") {
                    analysis.Services = append(analysis.Services, ServiceInfo{
                        Name: name,
                    })
                }
            }
        }
        return true
    })
    
    return nil
}

func analyzeProtoFile(path string, analysis *ProjectAnalysis) error {
    content, err := os.ReadFile(path)
    if err != nil {
        return err
    }
    
    // Simple proto parsing (use proper parser for production)
    lines := strings.Split(string(content), "\n")
    for _, line := range lines {
        if strings.Contains(line, "rpc ") {
            // Extract RPC definition
            parts := strings.Fields(line)
            if len(parts) >= 4 {
                analysis.APIs = append(analysis.APIs, APIInfo{
                    Method: parts[1],
                })
            }
        }
    }
    
    return nil
}

func main() {
    analysis, err := AnalyzeProject(".")
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("Project: %s\n", analysis.Name)
    fmt.Printf("Services: %d\n", len(analysis.Services))
    fmt.Printf("APIs: %d\n", len(analysis.APIs))
}
```

### Step 3: Generate Report

```go
// report.go
package main

import (
    "fmt"
    "html/template"
    "os"
)

const reportTemplate = `
# Project Analysis Report

## Overview
- **Project Name**: {{.Name}}
- **Services**: {{len .Services}}
- **APIs**: {{len .APIs}}

## Services
{{range .Services}}
- {{.Name}}
  {{range .Methods}}  - {{.}}
  {{end}}
{{end}}

## APIs
{{range .APIs}}
- {{.Service}}.{{.Method}}
  - Input: {{.Input}}
  - Output: {{.Output}}
{{end}}

## Recommendations
{{range .Recommendations}}
- {{.}}
{{end}}
`

func GenerateReport(analysis *ProjectAnalysis, outputPath string) error {
    // Add recommendations
    analysis.Recommendations = generateRecommendations(analysis)
    
    tmpl, err := template.New("report").Parse(reportTemplate)
    if err != nil {
        return err
    }
    
    file, err := os.Create(outputPath)
    if err != nil {
        return err
    }
    defer file.Close()
    
    return tmpl.Execute(file, analysis)
}

func generateRecommendations(analysis *ProjectAnalysis) []string {
    var recommendations []string
    
    if len(analysis.Services) == 0 {
        recommendations = append(recommendations, "No services found. Consider creating service layer.")
    }
    
    if len(analysis.APIs) == 0 {
        recommendations = append(recommendations, "No APIs found. Define proto files.")
    }
    
    return recommendations
}
```

## Usage

### Command Line

```bash
# Analyze project
./analyze-project.sh

# Generate report
go run analyze.go report.go
```

### In Claude Code

```
You: Analyze this kratos project

Claude: I'll analyze the project structure...

1. Project Structure:
   - api/: 3 proto files
   - internal/biz/: 2 usecases
   - internal/data/: 2 repositories
   - internal/service/: 2 services

2. APIs:
   - UserService: CreateUser, GetUser, UpdateUser, DeleteUser
   - OrderService: CreateOrder, GetOrder, ListOrders

3. Recommendations:
   - Add input validation middleware
   - Implement caching for GetUser
   - Add metrics collection
```

## Output Example

```markdown
# Project Analysis Report

## Overview
- **Project Name**: user-service
- **Services**: 2
- **APIs**: 7

## Services
- UserService
  - CreateUser
  - GetUser
  - UpdateUser
  - DeleteUser
  
- OrderService
  - CreateOrder
  - GetOrder
  - ListOrders

## APIs
- User.CreateUser
  - Input: CreateUserRequest
  - Output: CreateUserReply

## Recommendations
- Add input validation middleware
- Implement caching for GetUser
- Add metrics collection
```
