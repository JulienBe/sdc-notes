Where are IDs coming from for

```
POLICY-500I INFO: VF_MODULE_ARTIFACT: [
  {
    "vfModuleModelName": "VfVsimServiceHelmType2..vsim..module-0",
    "vfModuleModelInvariantUUID": "bf8a0ecf-6ab4-48f9-b3f4-d144ceb0ba5a",
    "vfModuleModelVersion": "1",
    "vfModuleModelUUID": "1b882af9-a0ef-4af1-8e4b-39520086ef90",
    "vfModuleModelCustomizationUUID": "d8dcfb45-b464-4570-ae1f-ae9dede0ede3",
    "isBase": true,
    "artifacts": [
      "76302027-5513-41e2-b17d-cf976550709b",
      "a8815860-fa73-4c8b-be7d-8ca63d80f798",
      "8399d492-b96e-4fca-ac8f-654935620e98"
    ],
    "properties": {
      "min_vf_module_instances": "1",
      "vf_module_label": "vsim",
      "max_vf_module_instances": "1",
      "vfc_list": "",
      "vf_module_description": "",
      "vf_module_type": "Base",
      "availability_zone_count": "",
      "volume_group": "false",
      "initial_count": "1"
    }
  }
]
```

```java
public class VfModuleArtifactPayloadEx {
	private String vfModuleModelName, vfModuleModelInvariantUUID, vfModuleModelVersion, vfModuleModelUUID, vfModuleModelCustomizationUUID, vfModuleModelDescription;
	private Boolean isBase;
	private List<String> artifacts;
	private Map< String, Object> properties;
}
```

Seems to be translated to

```java
public class VfModuleArtifactPayload {

    private String vfModuleModelName, vfModuleModelInvariantUUID, vfModuleModelVersion, vfModuleModelUUID, vfModuleModelCustomizationUUID, vfModuleModelDescription;
    private Boolean isBase;
    private List<String> artifacts;
    private Map< String, Object> properties;

    public VfModuleArtifactPayload(GroupDefinition group) {
        ...
        vfModuleModelInvariantUUID = group.getInvariantUUID();
        vfModuleModelUUID = group.getGroupUUID();
        artifacts = group.getArtifactsUuid();        
        ...
    }

    public VfModuleArtifactPayload(GroupInstance group) {
        ...
        vfModuleModelInvariantUUID = group.getInvariantUUID();
        vfModuleModelUUID = group.getGroupUUID();
        vfModuleModelCustomizationUUID = group.getCustomizationUUID();        
        artifacts = new ArrayList<>(group.getArtifactsUuid() != null ? group.getArtifactsUuid() : new LinkedList<>());
        artifacts.addAll(group.getGroupInstanceArtifactsUuid() != null ? group.getGroupInstanceArtifactsUuid() : new LinkedList<>());
        ...
    }
    ...
}
```

Used in `ArtifactUuidFix.java` 1300 lines & `ServiceBusinessLogic.java` 2569 lines

```java
@org.springframework.stereotype.Component("artifactUuidFix")
@Autowired
public class ArtifactUuidFix {    
    private JanusGraphDao janusGraphDao;
    private ToscaOperationFacade toscaOperationFacade;
    private ToscaExportHandler toscaExportUtils;
    private ArtifactCassandraDao artifactCassandraDao;
    private CsarUtils csarUtils;

```
