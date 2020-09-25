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

Seems to be translated to

```java
public class VfModuleArtifactPayloadEx {
	private String vfModuleModelName, vfModuleModelInvariantUUID, vfModuleModelVersion, vfModuleModelUUID, vfModuleModelCustomizationUUID, vfModuleModelDescription;
	private Boolean isBase;
	private List<String> artifacts;
	private Map< String, Object> properties;
}
```

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
    ...
    
    // Based on the name of this rather complicated method, probably not there: 
        private boolean isProblematicService(Service service, String serviceName) {

        List<ComponentInstance> componentInstances = service.getComponentInstances();

        if (componentInstances == null) {
            log.info("No instances for service {} ", service.getUniqueId());
            return false;
        }
        boolean isCheckVFModules = true;
        if (service.getLifecycleState() == LifecycleStateEnum.NOT_CERTIFIED_CHECKIN ||
                service.getLifecycleState() == LifecycleStateEnum.NOT_CERTIFIED_CHECKOUT) {
            isCheckVFModules = false;
        }
        for (ComponentInstance ci : componentInstances) {
            Map<String, ArtifactDefinition> deploymentArtifacts = ci.getDeploymentArtifacts();
            List<GroupInstance> groupInstances = ci.getGroupInstances();
            if (groupInstances == null || groupInstances.isEmpty()) {
                log.info("No instance groups for instance {} in service {} id {} ", ci.getName(), serviceName,
                        service.getUniqueId());
                continue;
            }
            List<VfModuleArtifactPayloadEx> vfModules = null;
            if (isCheckVFModules) {
                Optional<ArtifactDefinition> optionalVfModuleArtifact = deploymentArtifacts.values().stream()
                        .filter(p -> p.getArtifactType().equals(ArtifactTypeEnum.VF_MODULES_METADATA.getType())).findAny();

                if (!optionalVfModuleArtifact.isPresent())
                    return true;

                ArtifactDefinition vfModuleArtifact = optionalVfModuleArtifact.get();
                Either<List<VfModuleArtifactPayloadEx>, StorageOperationStatus> vfModulesEither = parseVFModuleJson(vfModuleArtifact);
                if (vfModulesEither.isRight()) {
                    log.error("Failed to parse vfModule for service {} status is {}", service.getUniqueId(), vfModulesEither.right().value());
                    return true;
                }
                vfModules = vfModulesEither.left().value();
                if (vfModules == null || vfModules.isEmpty()) {
                    log.info("vfModules empty for service {}", service.getUniqueId());
                    return true;
                }
            }

            for (GroupInstance gi : groupInstances) {
                if (gi.getType().equals(Constants.DEFAULT_GROUP_VF_MODULE)) {
                    VfModuleArtifactPayloadEx vfModule = null;
                    if (isCheckVFModules && vfModules != null) {
                        Optional<VfModuleArtifactPayloadEx> op = vfModules.stream().filter(vf -> vf.getVfModuleModelName().equals(gi.getGroupName())).findAny();
                        if (!op.isPresent()) {
                            log.error("Failed to find vfModule for group {}", gi.getGroupName());
                            return true;
                        }
                        vfModule = op.get();
                    }
                    if (isProblematicGroupInstance(gi, ci.getName(), serviceName, deploymentArtifacts, vfModule)) {
                        return true;
                    }
                }
            }

        }
        return false;
    }
    // Based on the name of this complicated method, probably not there either
       private boolean isProblematicGroupInstance(GroupInstance gi, String instName, String servicename,
                                               Map<String, ArtifactDefinition> deploymentArtifacts, VfModuleArtifactPayloadEx vfModule) {
        List<String> artifacts = gi.getArtifacts();
        List<String> artifactsUuid = gi.getArtifactsUuid();
        List<String> instArtifactsUuid = gi.getGroupInstanceArtifactsUuid();
        List<String> instArtifactsId = gi.getGroupInstanceArtifacts();
        Set<String> instArtifatIdSet = new HashSet<>();
        Set<String> artifactsSet = new HashSet<>();

        log.info("check group {} for instance {} ", gi.getGroupName(), instName);
        if ((artifactsUuid == null || artifactsUuid.isEmpty()) && (artifacts == null || artifacts.isEmpty())) {
            log.info("No instance groups for instance {} in service {} ", instName, servicename);
            return true;
        }
        artifactsSet.addAll(artifacts);
        if (artifactsSet.size() < artifacts.size()) {
            log.info(" artifactsSet.size() < artifacts.size() group {} in resource {} ", instName, servicename);
            return true;
        }

        if (instArtifactsId != null && !instArtifactsId.isEmpty()) {
            instArtifatIdSet.addAll(instArtifactsId);
        }

        if ((artifactsUuid != null) && (artifacts.size() < artifactsUuid.size())) {
            log.info(" artifacts.size() < artifactsUuid.size() inst {} in service {} ", instName, servicename);
            return true;
        }
        if (!artifacts.isEmpty() && (artifactsUuid == null || artifactsUuid.isEmpty())) {
            log.info(
                    " artifacts.size() > 0 && (artifactsUuid == null || artifactsUuid.isEmpty() inst {} in service {} ",
                    instName, servicename);
            return true;
        }
        if (artifactsUuid != null && artifactsUuid.contains(null)) {
            log.info(" artifactsUuid.contains(null) inst {} in service {} ", instName, servicename);
            return true;
        }
        if (instArtifactsId != null && instArtifatIdSet.size() < instArtifactsId.size()) {
            log.info(" instArtifatIdSet.size() < instArtifactsId.size() inst {} in service {} ", instName, servicename);
            return true;
        }

        if ((instArtifactsId != null && instArtifactsUuid != null)
                && instArtifactsId.size() != instArtifactsUuid.size()) {
            log.info(" instArtifactsId.size() != instArtifactsUuid.size() inst {} in service {} ", instName,
                    servicename);
            return true;
        }

        for (String artifactId : artifacts) {
            String artifactlabel = findArtifactLabelFromArtifactId(artifactId);
            ArtifactDefinition artifactDefinition = deploymentArtifacts.get(artifactlabel);
            if (artifactDefinition == null) {
                log.info(" artifactDefinition == null label {} inst {} in service {} ", artifactlabel, instName,
                        servicename);
                return true;
            }
            ArtifactTypeEnum artifactType = ArtifactTypeEnum.parse(artifactDefinition.getArtifactType());
            if (artifactType != ArtifactTypeEnum.HEAT_ENV) {
                if (!artifactId.equals(artifactDefinition.getUniqueId())) {
                    log.info(
                            " !artifactId.equals(artifactDefinition.getUniqueId() artifact {}  artId {} inst {} in service {} ",
                            artifactlabel, artifactId, instName, servicename);
                    return true;
                }
                if (artifactsUuid != null && !artifactsUuid.contains(artifactDefinition.getArtifactUUID())) {
                    log.info(
                            " artifactsUuid.contains(artifactDefinition.getArtifactUUID() label {} inst {} in service {} ",
                            artifactlabel, instName, servicename);
                    return true;
                }
            } else {
                if (instArtifactsUuid == null || instArtifactsUuid.isEmpty()) {
                    log.info(" instArtifactsUuid empty. label {} inst {} in service {} ", artifactlabel, instName,
                            servicename);
                    return true;
                }
                if (!instArtifactsUuid.contains(artifactDefinition.getArtifactUUID())) {
                    log.info(
                            " instArtifactsUuid.contains(artifactDefinition.getArtifactUUID() label {} inst {} in service {} ",
                            artifactlabel, instName, servicename);
                    return true;
                }
            }
        }
        for (String artifactUUID : artifactsUuid) {
            String label = findArtifactLabelFromArtifactId(artifactUUID);
            if (label != null && !label.isEmpty() && !"".equals(label)) {
                return true;
            }
        }
        if (vfModule != null && artifactsUuid != null) {
            return isProblematicVFModule(vfModule, artifactsUuid, instArtifactsUuid);
        }
        return false;
    }
    // same here, I'm guessing that checking if it's 'problematic' has nothing to do with generating ids
        private boolean isProblematicVFModule(VfModuleArtifactPayloadEx vfModule, List<String> artifactsUuid,
                                          List<String> instArtifactsUuid) {
        log.info(" isProblematicVFModule  {}  ", vfModule.getVfModuleModelName());
        List<String> vfModuleArtifacts = vfModule.getArtifacts();
        List<String> allArtifacts = new ArrayList<>();
        allArtifacts.addAll(artifactsUuid);
        if (instArtifactsUuid != null)
            allArtifacts.addAll(instArtifactsUuid);
        if ((vfModuleArtifacts == null || vfModuleArtifacts.isEmpty()) && !artifactsUuid.isEmpty()) {
            log.error(" vfModuleArtifacts == null || vfModuleArtifacts.isEmpty()) && !artifactsUuid.isEmpty()");
            return true;
        }
        if (vfModuleArtifacts != null) {
            if (vfModuleArtifacts.size() != allArtifacts.size()) {
                log.error(" vfModuleArtifacts.size() != allArtifacts.size()");
                return true;
            }
            for (String vfModuleArtifact : vfModuleArtifacts) {
                Optional<String> op = allArtifacts.stream().filter(a -> a.equals(vfModuleArtifact)).findAny();
                if (!op.isPresent()) {
                    log.error("failed to find artifact {} in group artifacts {}", vfModuleArtifact, allArtifacts);
                    return true;
                }
            }
        }
        return false;
    }
    
        private void fixComponentInstances(Service service, ComponentInstance instance) {
        Map<String, ArtifactDefinition> artifactsMap = instance.getDeploymentArtifacts();
        List<GroupInstance> groupsList = instance.getGroupInstances();
        if (groupsList != null && artifactsMap != null) {
            List<GroupInstance> groupsToDelete = new ArrayList<>();
            for (GroupInstance group : groupsList) {
                fixGroupInstances(service, artifactsMap, groupsToDelete, group);

            }

            if (!groupsToDelete.isEmpty()) {
                log.debug("Migration1707ArtifactUuidFix  delete group:  resource id {}, group instance to delete {} ",
                        service.getUniqueId(), groupsToDelete);
                groupsList.removeAll(groupsToDelete);

            }

            Optional<ArtifactDefinition> optionalVfModuleArtifact = artifactsMap.values().stream()
                    .filter(p -> p.getArtifactType().equals(ArtifactTypeEnum.VF_MODULES_METADATA.getType())).findAny();
            ArtifactDefinition vfModuleArtifact;
            if (!optionalVfModuleArtifact.isPresent()) {
                vfModuleArtifact = createVfModuleArtifact(instance, service);
                artifactsMap.put(vfModuleArtifact.getArtifactLabel(), vfModuleArtifact);
            } else {
                vfModuleArtifact = optionalVfModuleArtifact.get();
            }
            fillVfModuleInstHeatEnvPayload(service, instance, groupsList, vfModuleArtifact);
        }
    }
    // The 'artifact definition' doesn't carry any notion of id ? Service unused
        private ArtifactDefinition createVfModuleArtifact(ComponentInstance currVF, Service service) {

        ArtifactDefinition vfModuleArtifactDefinition = new ArtifactDefinition();

        vfModuleArtifactDefinition.setDescription("Auto-generated VF Modules information artifact");
        vfModuleArtifactDefinition.setArtifactDisplayName("Vf Modules Metadata");
        vfModuleArtifactDefinition.setArtifactType(ArtifactTypeEnum.VF_MODULES_METADATA.getType());
        vfModuleArtifactDefinition.setArtifactGroupType(ArtifactGroupTypeEnum.DEPLOYMENT);
        vfModuleArtifactDefinition.setArtifactLabel("vfModulesMetadata");
        vfModuleArtifactDefinition.setTimeout(0);
        vfModuleArtifactDefinition.setArtifactName(currVF.getNormalizedName() + "_modules.json");

        return vfModuleArtifactDefinition;
    }
    // talking about ids here at least !
        private void fillVfModuleInstHeatEnvPayload(Component parent, ComponentInstance instance, List<GroupInstance> groupsForCurrVF,
                                                ArtifactDefinition vfModuleArtifact) {
        log.debug("generate new vf module for component. name  {}, id {}, Version {}", instance.getName(), instance.getUniqueId());

        String uniqueId = UniqueIdBuilder.buildInstanceArtifactUniqueId(parent.getUniqueId(), instance.getUniqueId(), vfModuleArtifact.getArtifactLabel());

        vfModuleArtifact.setUniqueId(uniqueId);
        vfModuleArtifact.setEsId(vfModuleArtifact.getUniqueId());

        List<VfModuleArtifactPayload> vfModulePayloadForCurrVF = new ArrayList<>();
        if (groupsForCurrVF != null) {
            for (GroupInstance groupInstance : groupsForCurrVF) {
                VfModuleArtifactPayload modulePayload = new VfModuleArtifactPayload(groupInstance);
                vfModulePayloadForCurrVF.add(modulePayload);
            }
            Collections.sort(vfModulePayloadForCurrVF,
                    (art1, art2) -> VfModuleArtifactPayload.compareByGroupName(art1, art2));
            final Gson gson = new GsonBuilder().setPrettyPrinting().create();

            String vfModulePayloadString = gson.toJson(vfModulePayloadForCurrVF);
            log.debug("vfModulePayloadString {}", vfModulePayloadString);
            if (vfModulePayloadString != null) {
                String newCheckSum = GeneralUtility
                        .calculateMD5Base64EncodedByByteArray(vfModulePayloadString.getBytes());
                vfModuleArtifact.setArtifactChecksum(newCheckSum);

                DAOArtifactData artifactData = new DAOArtifactData(vfModuleArtifact.getEsId(),
                        vfModulePayloadString.getBytes());
                artifactCassandraDao.saveArtifact(artifactData);
            }
        }
    }
    // parsing a json
        private Either<List<VfModuleArtifactPayloadEx>, StorageOperationStatus> parseVFModuleJson(ArtifactDefinition vfModuleArtifact) {
        log.info("Try to get vfModule json from cassandra {}", vfModuleArtifact.getEsId());
        Either<DAOArtifactData, CassandraOperationStatus> vfModuleData = artifactCassandraDao.getArtifact(vfModuleArtifact.getEsId());

        if (vfModuleData.isRight()) {
            CassandraOperationStatus resourceUploadStatus = vfModuleData.right().value();
            StorageOperationStatus storageResponse = DaoStatusConverter.convertCassandraStatusToStorageStatus(resourceUploadStatus);
            log.error("failed to fetch vfModule json {} from cassandra. Status is {}", vfModuleArtifact.getEsId(), storageResponse);
            return Either.right(storageResponse);

        }

        DAOArtifactData DAOArtifactData = vfModuleData.left().value();
        String gsonData = new String(DAOArtifactData.getDataAsArray());
        final Gson gson = new GsonBuilder().setPrettyPrinting().create();
        JsonArray jsonElement = new JsonArray();
        jsonElement = gson.fromJson(gsonData, jsonElement.getClass());
        List<VfModuleArtifactPayloadEx> vfModules = new ArrayList<>();
        jsonElement.forEach(je -> {
            VfModuleArtifactPayloadEx vfModule = ComponentsUtils.parseJsonToObject(je.toString(), VfModuleArtifactPayloadEx.class);
            vfModules.add(vfModule);
        });

        log.debug("parse vf module finish {}", gsonData);
        return Either.left(vfModules);

    }
```
