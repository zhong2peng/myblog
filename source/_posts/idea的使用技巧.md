---
title: idea的使用技巧
date: 2019-06-05 11:37:59
tags:
---
# 自动生成JPA的实体类
脚本可以合并，一次性生成所需的类，也可以进行改写，适配于自己的项目
## Generate POJOs.groovy
```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */

packageName = "com.dnight.base.framework.entity;"    // 此处指定包路径，也就对应实体类中的package com.topex.admin.entity;
typeMapping = [    // 此处指定对应的类型映射，如下：数据库中bigint对应生成java的Long，int|tinyint生成Integer...
                   (~/(?i)bigint/)                      : "Long",
                   (~/(?i)int|tinyint/)                 : "Integer",
                   (~/(?i)float|double|decimal|real/): "BigDecimal",
                   (~/(?i)date|datetime|timestamp/)       : "Date",
                   (~/(?i)time/)                     : "Time",
                   (~/(?i)/)                         : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable }.each { generate(it, dir) }
}

def generate(table, dir) {
    def className = javaName(table.getName(), true, true)
    def fields = calcFields(table)
    new PrintWriter(new OutputStreamWriter(new FileOutputStream( new File(dir, className + ".java")), "utf-8")).withPrintWriter { out -> generate(out, className, fields, table.getName()) }
}

def generate(out, className, fields, tableName) {   // 从这里开始，拼实体类的具体逻辑代码
    out.println "package $packageName"
    out.println ""
    out.println ""
    out.println "import lombok.*;"     // 因为我使用了lombok插件，使用到了Data注解，所以在引包时加了这一行
    out.println "import io.swagger.annotations.ApiModelProperty;"   // 同上，使用了swagger文档，所以引入到需要的注解
    out.println "import javax.persistence.*;"  // tk.mybatis插件需用时需要@id注解，所以引入，不需要就去掉
    out.println "import java.io.Serializable;"
    out.println ""
    out.println "@Data"
    out.println "@Builder"
    out.println "@AllArgsConstructor"
    out.println "@NoArgsConstructor"
    out.println "@Entity"
    out.println "@Table(name = \"$tableName\")"
    out.println "public class $className implements Serializable {"
    out.println ""
    out.println "private static final long serialVersionUID = 1L;"
    out.println ""
    int i = 0
    fields.each() {   // 遍历字段，按下面的规则生成
        // 输出注释，这里唯一的是id特殊判断了一下，如果判断it.name == id, 则多添加一行@Id
        if (it.name == "id") {
            if (!isNotEmpty(it.commoent)) {
                out.println "\t/**"
                out.println "\t * 主键id"
                out.println "\t */"
                out.println "\t@ApiModelProperty(value = \"主键id\", position = ${i})"
            }
            out.println "\t@Id"
            out.println "\t@GeneratedValue(strategy = GenerationType.IDENTITY)"
        }
        if (isNotEmpty(it.commoent)) {
            out.println "\t/**"
            out.println "\t * ${it.commoent}"
            out.println "\t */"
            out.println "\t@ApiModelProperty(value = \"${it.commoent}\")"
        }
        if (it.annos != "")
            out.println "\t@Column(name = \"${it.annos}\")"
        out.println "\tprivate ${it.type} ${it.name};"
        out.println ""
        i++
    }
    out.println ""
    out.println "}"
}

def calcFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())
        def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
        fields += [[
                           name : javaName(col.getName(), false, false),
                           type : typeStr,
                           commoent: col.getComment(),
                           annos: col.getName()]]
    }
}

def isNotEmpty(content) {
    return content != null && content.toString().trim().length() > 0
}

def javaName(str, capitalize, flag) {
    if (!flag){
        def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
                .collect { Case.LOWER.apply(it).capitalize() }
                .join("")
                .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
        name = capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
    }else{
        def sTmp = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
        sTmp[0] = ""
        def s = sTmp.collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
        name = capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
    }

}
```
## Generate Daos
```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */

packageName = "com.dnight.base.framework.entity;" // 需手动配置 生成的 dao 所在包位置

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable }.each { generate(it, dir) }
}

def generate(table, dir) {
    def baseName = javaName(table.getName(), true)
    new File(dir, baseName + "Repository.java").withPrintWriter { out -> generateInterface(out, baseName) }
}

def generateInterface(out, baseName) {
    def date = new Date().format("yyyy/MM/dd")
    out.println "package $packageName"
//    out.println "import cn.xx.entity.${baseName}Entity;" // 需手动配置
    out.println "import org.springframework.data.jpa.repository.JpaRepository;"
    out.println "import org.springframework.data.jpa.repository.JpaSpecificationExecutor;"
    out.println "import org.springframework.stereotype.Repository;"
    out.println ""
    out.println "/**"
    out.println " * Created on $date."
    out.println " *"
    out.println " * @author zhongpeng" // 可自定义
    out.println " */"
    out.println "@Repository"
    out.println "public interface ${baseName}Repository extends JpaRepository<${baseName}, Long>, JpaSpecificationExecutor<${baseName}> {" // 可自定义
    out.println ""
    out.println "}"
}

def javaName(str, capitalize) {
//    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
//            .collect { Case.LOWER.apply(it).capitalize() }
//            .join("")
//            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    def sTmp = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
    sTmp[0] = ""
    def s = sTmp.collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    name = capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
}
```
## Generate Service
```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */

packageName = "com.dnight.base.framework.entity;" // 需手动配置 生成的 dao 所在包位置

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable }.each { generate(it, dir) }
}

def generate(table, dir) {
    def baseName = javaName(table.getName(), true)
    new File(dir, baseName + "Service.java").withPrintWriter { out -> generateInterface(out, baseName) }
}

def generateInterface(out, baseName) {
    def date = new Date().format("yyyy/MM/dd")
    out.println "package $packageName"
//    out.println "import cn.xx.entity.${baseName}Entity;" // 需手动配置
    out.println ""
    out.println "/**"
    out.println " * Created on $date."
    out.println " *"
    out.println " * @author zhongpeng" // 可自定义
    out.println " */"
    out.println "public interface ${baseName}Service {" // 可自定义
    out.println ""
    out.println "}"
}

def javaName(str, capitalize) {
//    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
//            .collect { Case.LOWER.apply(it).capitalize() }
//            .join("")
//            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    def sTmp = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
    sTmp[0] = ""
    def s = sTmp.collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    name = capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
}
```
## Generate Controllers
```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */

packageName = "com.dnight.base.framework.entity;" // 需手动配置 生成的 dao 所在包位置

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable }.each { generate(it, dir) }
}

def generate(table, dir) {
    def baseName = javaName(table.getName(), true)
    new File(dir, baseName + "Controller.java").withPrintWriter { out -> generateInterface(out, baseName) }
}

def generateInterface(out, baseName) {
    def date = new Date().format("yyyy/MM/dd")
    out.println "package $packageName"
//    out.println "import cn.xx.entity.${baseName}Entity;" // 需手动配置
    out.println "import io.swagger.annotations.Api;"
    out.println "import io.swagger.annotations.ApiOperation;"
    out.println "import org.slf4j.Logger;"
    out.println "import org.slf4j.LoggerFactory;"
    out.println "import org.springframework.web.bind.annotation.*;"
    out.println ""
    out.println "/**"
    out.println " * Created on $date."
    out.println " *"
    out.println " * @author zhongpeng" // 可自定义
    out.println " */"
    out.println "@RestController"
    out.println "@RequestMapping(value = WebConstants.WEB_PREFIX + \"/xx\")"
    out.println "@Api(tags = \"xx\", description = \"xx\")"
    out.println "public class ${baseName}Controller {" // 可自定义
    out.println ""
    out.println "\tprivate static final Logger LOGGER = LoggerFactory.getLogger(${baseName}Controller.class);"
    out.println ""
    out.println "}"
}

def javaName(str, capitalize) {
//    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
//            .collect { Case.LOWER.apply(it).capitalize() }
//            .join("")
//            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    def sTmp = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
    sTmp[0] = ""
    def s = sTmp.collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    name = capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
}
```
## Generate DTOs
```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */

packageName = "com.dnight.base.framework.entity;"    // 此处指定包路径，也就对应实体类中的package com.topex.admin.entity;
typeMapping = [    // 此处指定对应的类型映射，如下：数据库中bigint对应生成java的Long，int|tinyint生成Integer...
                   (~/(?i)bigint/)                      : "Long",
                   (~/(?i)int|tinyint/)                 : "Integer",
                   (~/(?i)float|double|decimal|real/): "BigDecimal",
                   (~/(?i)date|datetime|timestamp/)       : "Date",
                   (~/(?i)time/)                     : "Time",
                   (~/(?i)/)                         : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable }.each { generate(it, dir) }
}

def generate(table, dir) {
    def className = javaName(table.getName(), true, true)
    def fields = calcFields(table)
    new PrintWriter(new OutputStreamWriter(new FileOutputStream( new File(dir, className + "DTO.java")), "utf-8")).withPrintWriter { out -> generate(out, className, fields, table.getName()) }
}

def generate(out, className, fields, tableName) {   // 从这里开始，拼实体类的具体逻辑代码
    out.println "package $packageName"
    out.println ""
    out.println ""
    out.println "import lombok.*;"     // 因为我使用了lombok插件，使用到了Data注解，所以在引包时加了这一行
    out.println "import io.swagger.annotations.ApiModelProperty;"   // 同上，使用了swagger文档，所以引入到需要的注解
    out.println "import java.io.Serializable;"
    out.println ""
    out.println "@Data"
    out.println "@Builder"
    out.println "@AllArgsConstructor"
    out.println "@NoArgsConstructor"
    out.println "public class ${className}DTO implements Serializable {"
    out.println ""
    out.println "private static final long serialVersionUID = 1L;"
    out.println ""
    int i = 0
    fields.each() {   // 遍历字段，按下面的规则生成
        // 输出注释，这里唯一的是id特殊判断了一下，如果判断it.name == id, 则多添加一行@Id
        if (it.name == "id") {
            if (!isNotEmpty(it.commoent)) {
                out.println "\t/**"
                out.println "\t * 主键id"
                out.println "\t */"
                out.println "\t@ApiModelProperty(value = \"主键id\", position = ${i})"
            }
        }
        if (isNotEmpty(it.commoent)) {
            out.println "\t/**"
            out.println "\t * ${it.commoent}"
            out.println "\t */"
            out.println "\t@ApiModelProperty(value = \"${it.commoent}\")"
        }
        out.println "\tprivate ${it.type} ${it.name};"
        out.println ""
        i++
    }
    out.println ""
    out.println "}"
}

def calcFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())
        def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
        fields += [[
                           name : javaName(col.getName(), false, false),
                           type : typeStr,
                           commoent: col.getComment(),
                           annos: col.getName()]]
    }
}

def isNotEmpty(content) {
    return content != null && content.toString().trim().length() > 0
}

def javaName(str, capitalize, flag) {
    if (!flag){
        def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
                .collect { Case.LOWER.apply(it).capitalize() }
                .join("")
                .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
        name = capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
    }else{
        def sTmp = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
        sTmp[0] = ""
        def s = sTmp.collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
        name = capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
    }

}
```

