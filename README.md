# mb-pipelines-openapi-generator

Pipeline to generate clients and mocks based on openapi specification using the openapi generator.
For more information:
- https://github.com/OpenAPITools/openapi-generator/tree/master
- https://github.com/OpenAPITools/openapi-generator/blob/master/docs/generators/java.md

## local usage
First download the jar using the [openapi readme](https://github.com/OpenAPITools/openapi-generator/tree/master?tab=readme-ov-file#13---download-jar).
example commands:

### local debugging
```
java -jar openapi-generator-cli-7.15.0.jar generate -i openapi.yaml --template-dir supporting-files/generator-templates/java/jersey3  -c supporting-files/generator-configs/java-jersey3.yaml -o ./generated-client
```

Output json model for operations to use in template files
```
java -jar openapi-generator-cli-7.15.0.jar generate -i openapi.yaml --template-dir supporting-files/generator-templates/java/jersey3  -c supporting-files/generator-configs/java-jersey3.yaml -o ./generated-client --global-property debugOperations=true 
```

### mock
```
java -jar openapi-generator-cli-7.15.0.jar generate -i oas-files/out-directapply-rone-v2.yaml -g java-wiremock -o ./output/out-directapply-rone-v2 --api-name-suffix apiClient --invoker-package nl.randstadgroep.xone.ta.client.directapply.rone.v2.client --api-package nl.randstadgroep.xone.ta.client.directapply.rone.v2.api --model-package nl.randstadgroep.xone.ta.client.directapply.rone.v2.model --additional-properties=asyncNative=false,groupId=nl.randstadgroep.xone.ta.client,artifactId=out-directapply-rone-api-client-v2,artifactVersion=2.0.0-ba680182-SNAPSHOT,licenseName=OwnedByRandstad,licenseUrl=https://randstadgroep.nl/,developerName=NLTestAutomation,developerEmail=xone.nl.test.automation@randstadgroep.nl,developerOrganization=randstadgroep,developerOrganizationUrl=https://randstadgroep.nl/,dateLibrary=java8,useJakartaEe=true,openApiNullable=false,serializableModel=true,serializationLibrary=jackson,generateClientAsBean=true,library=native,legacyDiscriminatorBehavior=true
```