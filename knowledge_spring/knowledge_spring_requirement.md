# Knowledge Spring - Organization Plan Requirement

## Requirement Confirmation

You want me to:
1. Review all files in the `knowledge_spring` directory carefully
2. Based on the contents of those files, provide a plan for organizing them into:
   - **Claude Skills**: Reusable knowledge that should be available for reference
   - **Claude Sub-agents**: Specialized agents that can be invoked for specific tasks
   - **MCPs (Model Context Protocol)**: Tools/servers that provide capabilities
3. Base the categorization on the actual content and purpose of each file

## Summary of Files Reviewed

I've reviewed all the guideline files in the `knowledge_spring` directory:

- **`index.md`** - Navigation and overview guide
- **`GuideLine_1_general_style.md`** - General coding standards, principles, and workflow
- **`GuideLine_2_javacore.md`** - Java 21 language features (records, sealed classes, pattern matching)
- **`GuideLine_3_functionnal.md`** - Functional programming patterns
- **`GuideLine_4_package_ddd.md`** - Modular monolith architecture with DDD patterns
- **`GuideLine_4_package_rich.md`** - Rich domain model architecture with full DDD implementation
- **`GuideLine_5_springeco.md`** - Spring Boot 3.x ecosystem guidelines
- **`GuideLine_6_api.md`** - REST Controller implementation and OpenAPI documentation
- **`GuideLine_7_testng.md`** - Testing guidelines using TestNG framework
- **`GuideLine_8_allchecklist.md`** - Quick reference cheat sheet and checklists
- **`refactoring.md`** - Refactoring guidelines
- **`design-patterns.md`** - Design pattern preferences
- **`decisions.md`** - Architectural Decision Records (currently empty)

## Proposed Organization Plan

### 1. Claude Skills (Reference Knowledge)
These should be available as reference knowledge that can be consulted during development:

- **`GuideLine_1_general_style.md`** - Core coding principles and standards
- **`GuideLine_2_javacore.md`** - Java 21 language features reference
- **`GuideLine_3_functionnal.md`** - Functional programming patterns
- **`GuideLine_5_springeco.md`** - Spring Boot ecosystem guidelines
- **`GuideLine_8_allchecklist.md`** - Quick reference and checklists
- **`refactoring.md`** - Refactoring guidelines
- **`design-patterns.md`** - Design pattern preferences
- **`decisions.md`** - Architectural Decision Records

### 2. Claude Sub-agents (Specialized Tasks)
These should be specialized agents that can be invoked for specific development tasks:

- **Architecture Design Agent**: 
  - `GuideLine_4_package_ddd.md` - Modular monolith architecture
  - `GuideLine_4_package_rich.md` - Rich domain model architecture
  
- **API Development Agent**: 
  - `GuideLine_6_api.md` - REST Controller and OpenAPI documentation
  
- **Testing Agent**: 
  - `GuideLine_7_testng.md` - Testing guidelines with TestNG

### 2. Claude Sub-agents (Specialized Tasks) - UPDATED
These should be specialized agents that can be invoked for specific development tasks:

- **Architecture Design Agent**: 
  - `GuideLine_4_package_ddd.md` - Modular monolith architecture
  - `GuideLine_4_package_rich.md` - Rich domain model architecture
  
- **API Development Agent**: 
  - `GuideLine_6_api.md` - REST Controller and OpenAPI documentation
  
- **Testing Agent**: 
  - `GuideLine_7_testng.md` - Testing guidelines with TestNG

- **Code Quality Validator Agent**: 
  - Validates code against all guidelines
  - Uses all guideline files as reference
  - Provides code review and quality checks

- **Architecture Validator Agent**: 
  - Enforces architectural rules
  - Validates layer dependencies
  - Checks compliance with DDD patterns

### 3. MCPs (Model Context Protocol - Tools/Capabilities)
These should be external tools/servers that provide capabilities (if any):

**Note:** MCPs would be external tools/services, not guideline-based validators. Examples might include:
- External linters or static analysis tools
- CI/CD integration tools
- External API services
- Database connection tools

**Question:** Do you have specific external tools/services in mind for MCPs, or should we focus on Skills and Sub-agents only?

---

**Please confirm if this matches your understanding, or let me know if you'd like me to adjust the categorization. Once confirmed, I'll provide the detailed plan with rationale for each categorization.**

