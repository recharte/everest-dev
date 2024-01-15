# Vars
everest_namespace = 'percona-everest'
namespaces_string = os.getenv('NAMESPACES', 'my-special-place,the-dark-side,percona-everest')
print('Using namespaces: %s' % namespaces_string)
pxc_operator_version = os.getenv('PXC_OPERATOR_VERSION', '1.13.0')
print('Using PXC operator version: %s' % pxc_operator_version)
psmdb_operator_version = os.getenv('PSMDB_OPERATOR_VERSION', '1.15.0')
print('Using PSMDB operator version: %s' % psmdb_operator_version)
pg_operator_version = os.getenv('PG_OPERATOR_VERSION', '2.2.0')
print('Using PG operator version: %s' % pg_operator_version)

# Check for required env vars
backend_dir = os.getenv('EVEREST_BACKEND_DIR')
if not backend_dir:
  fail('EVEREST_BACKEND_DIR must be set')
backend_dir = os.path.abspath(backend_dir)
if not os.path.exists(backend_dir):
  fail('Backend dir does not exist: %s' % backend_dir)
print('Using backend dir: %s' % backend_dir)

frontend_dir = os.getenv('EVEREST_FRONTEND_DIR')
if not frontend_dir:
  fail('EVEREST_FRONTEND_DIR must be set')
frontend_dir = os.path.abspath(frontend_dir)
if not os.path.exists(frontend_dir):
  fail('Frontend dir does not exist: %s' % frontend_dir)
print('Using frontend dir: %s' % frontend_dir)

operator_dir = os.getenv('EVEREST_OPERATOR_DIR')
if not operator_dir:
  fail('EVEREST_OPERATOR_DIR must be set')
operator_dir = os.path.abspath(operator_dir)
if not os.path.exists(operator_dir):
  fail('Operator dir does not exist: %s' % operator_dir)
print('Using operator dir: %s' % operator_dir)

# Ensure operator's kustomize is installed
local('[ -x %s/bin/kustomize ] || make -C %s kustomize' % (operator_dir, operator_dir), quiet=True)

# Ensure frontend repo is initialized
local('make -C %s init' % (frontend_dir), quiet=True)

# Create namespaces
load('ext://namespace', 'namespace_create', 'namespace_inject')
namespaces = namespaces_string.split(',')
for namespace in namespaces:
  namespace_create(namespace)
k8s_resource(
  objects=[
    '%s:namespace' % namespace for namespace in namespaces
  ],
  new_name='namespaces',
)

load('ext://configmap', 'configmap_from_dict')
k8s_yaml(configmap_from_dict('everest-configuration', everest_namespace, inputs={'namespaces': namespaces_string}))
k8s_resource(
  objects=[
    'everest-configuration:configmap'
  ],
  resource_deps = [
    'namespaces',
  ],
  new_name='everest-config',
)


#################################
## Install DB engine operators ##
#################################

# PXC
pxc_operator_bundle_yaml = local(['curl', '-s', 'https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v%s/deploy/bundle.yaml' % pxc_operator_version], quiet=True)
# We keep track of the CRD objects separately from the operator deployment so
# that we can make the deployment depend on the CRDs to avoid tilt thinking
# that there are conflicts with multiple CRDs.
k8s_resource(
  objects=[
    # The CRDs don't really get installed in a namespace, but since each
    # operator install is tied to a namespace, tilt uses a namespace to track
    # the install so we need to include one object per CRD per namespace.
    '%s:%s' % (crd, namespace) for namespace in namespaces for crd in [
            'perconaxtradbclusterbackups.pxc.percona.com:customresourcedefinition',
            'perconaxtradbclusterrestores.pxc.percona.com:customresourcedefinition',
            'perconaxtradbclusters.pxc.percona.com:customresourcedefinition'
        ]
  ],
  resource_deps = [
    'namespaces',
  ],
  new_name='pxc-crds',
  labels=["db-operators"]
)
for namespace in namespaces:
  k8s_yaml(namespace_inject(pxc_operator_bundle_yaml, namespace))
  k8s_resource(
    workload='percona-xtradb-cluster-operator:deployment:%s' % namespace,
    objects=[
      'percona-xtradb-cluster-operator:serviceaccount:%s' % namespace,
      'percona-xtradb-cluster-operator:role:%s' % namespace,
      'service-account-percona-xtradb-cluster-operator:rolebinding:%s' % namespace,
    ],
    resource_deps = [
      'namespaces',
      'pxc-crds',
    ],
    new_name='pxc:%s' % namespace,
    labels=["db-operators"]
  )

# PSMDB
psmdb_operator_bundle_yaml = local(['curl', '-s', 'https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/v%s/deploy/bundle.yaml' % psmdb_operator_version], quiet=True)
# We keep track of the CRD objects separately from the operator deployment so
# that we can make the deployment depend on the CRDs to avoid tilt thinking
# that there are conflicts with multiple CRDs.
k8s_resource(
  objects=[
    # The CRDs don't really get installed in a namespace, but since each
    # operator install is tied to a namespace, tilt uses a namespace to track
    # the install so we need to include one object per CRD per namespace.
    '%s:%s' % (crd, namespace) for namespace in namespaces for crd in [
      'perconaservermongodbbackups.psmdb.percona.com:customresourcedefinition',
      'perconaservermongodbrestores.psmdb.percona.com:customresourcedefinition',
      'perconaservermongodbs.psmdb.percona.com:customresourcedefinition',
    ]
  ],
  resource_deps = [
    'namespaces',
  ],
  new_name='psmdb-crds',
  labels=["db-operators"]
)
for namespace in namespaces:
  k8s_yaml(namespace_inject(psmdb_operator_bundle_yaml, namespace))
  k8s_resource(
    workload='percona-server-mongodb-operator:deployment:%s' % namespace,
    objects=[
      'percona-server-mongodb-operator:serviceaccount:%s' % namespace,
      'percona-server-mongodb-operator:role:%s' % namespace,
      'service-account-percona-server-mongodb-operator:rolebinding:%s' % namespace,
    ],
    resource_deps = [
      'namespaces',
      'psmdb-crds',
    ],
    new_name='psmdb:%s' % namespace,
    labels=["db-operators"]
  )

# PG
pg_operator_bundle_yaml = local(['curl', '-s', 'https://raw.githubusercontent.com/percona/percona-postgresql-operator/v%s/deploy/bundle.yaml' % pg_operator_version], quiet=True)
# We keep track of the CRD objects separately from the operator deployment so
# that we can make the deployment depend on the CRDs to avoid tilt thinking
# that there are conflicts with multiple CRDs.
k8s_resource(
  objects=[
    # The CRDs don't really get installed in a namespace, but since each
    # operator install is tied to a namespace, tilt uses a namespace to track
    # the install so we need to include one object per CRD per namespace.
    '%s:%s' % (crd, namespace) for namespace in namespaces for crd in [
      'perconapgbackups.pgv2.percona.com:customresourcedefinition',
      'perconapgclusters.pgv2.percona.com:customresourcedefinition',
      'perconapgrestores.pgv2.percona.com:customresourcedefinition',
      'postgresclusters.postgres-operator.crunchydata.com:customresourcedefinition',
    ]
  ],
  resource_deps = [
    'namespaces',
  ],
  new_name='pg-crds',
  labels=["db-operators"]
)
for namespace in namespaces:
  k8s_yaml(namespace_inject(pg_operator_bundle_yaml, namespace))
  k8s_resource(
    workload='percona-postgresql-operator:deployment:%s' % namespace,
    objects=[
      'percona-postgresql-operator:serviceaccount:%s' % namespace,
      'percona-postgresql-operator:role:%s' % namespace,
      'service-account-percona-postgresql-operator:rolebinding:%s' % namespace,
    ],
    resource_deps = [
      'namespaces',
      'pg-crds',
    ],
    new_name='pg:%s' % namespace,
    labels=["db-operators"]
  )

################################
####### Everest operator #######
################################

# Generate the Everest operator manifests
local_resource(
  'operator-gen-manifests',
  'make -C %s fmt manifests generate' % operator_dir,
  deps=['%s/api' % operator_dir],
  ignore=['%s/api/*/zz_generated.deepcopy.go' % operator_dir],
  labels=["everest-operator"]
)

# Build the Everest operator manager locally to take advantage of the go cache
local_resource(
  'operator-build',
  'make -C %s fmt; CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -C %s -o ./bin/manager ./main.go' % (operator_dir, operator_dir),
  deps=['%s/api' % operator_dir, '%s/controllers' % operator_dir, '%s/main.go' % operator_dir],
  labels=["everest-operator"]
)

# Live update the Everest operator manager without generating a new pod
load('ext://restart_process', 'docker_build_with_restart')
docker_build_with_restart('perconalab/everest-operator',
  context='%s' % operator_dir,
  dockerfile='./everest-operator.Dockerfile',
  entrypoint='/home/tilt/manager',
  only=['%s/bin/manager' % operator_dir],
  live_update=[
    sync('%s/bin/manager' % operator_dir, '/home/tilt/manager'),
  ]
)

# Apply Everest operator manifests
everest_operator_yaml=namespace_inject(kustomize('%s/config/default' % operator_dir, kustomize_bin='%s/bin/kustomize' % operator_dir), everest_namespace)
# Inject WATCH_NAMESPACES environment variable
objects = decode_yaml_stream(everest_operator_yaml)
for object in objects:
  if object.get('kind', None) == 'Deployment' and object.get('metadata', None).get('name', None) == 'everest-operator-controller-manager':
    for container in object['spec']['template']['spec']['containers']:
      if container.get('name') == 'manager':
        container['env'].append({'name': 'WATCH_NAMESPACES', 'value': 'dev'})
everest_operator_yaml = encode_yaml_stream(objects)
k8s_yaml(everest_operator_yaml)
k8s_resource(
  workload='everest-operator-controller-manager',
  objects=[
    'everest-operator-system:namespace',
    'backupstorages.everest.percona.com:customresourcedefinition',
    'databaseclusterbackups.everest.percona.com:customresourcedefinition',
    'databaseclusterrestores.everest.percona.com:customresourcedefinition',
    'databaseclusters.everest.percona.com:customresourcedefinition',
    'databaseengines.everest.percona.com:customresourcedefinition',
    'monitoringconfigs.everest.percona.com:customresourcedefinition',
    'everest-operator-controller-manager:serviceaccount',
    'everest-operator-leader-election-role:role',
    'everest-operator-manager-role:clusterrole',
    'everest-operator-metrics-reader:clusterrole',
    'everest-operator-proxy-role:clusterrole',
    'everest-operator-leader-election-rolebinding:rolebinding',
    'everest-operator-manager-rolebinding:clusterrolebinding',
    'everest-operator-proxy-rolebinding:clusterrolebinding',
  ],
  resource_deps = [
    '%s:%s' % (operator, namespace) for namespace in namespaces for operator in [
      'pxc',
      'psmdb',
      'pg',
    ]
  ],
  new_name='everest-operator',
  labels=["everest-operator"]
)

#################################
############ Everest ############
#################################

# Build backend
local_resource(
  'backend-build',
  'CGO_ENABLED=0 GOOS=linux GOARCH=amd64 make -C %s build' % backend_dir,
  deps=['%s/api' % backend_dir, '%s/cmd' % backend_dir, '%s/pkg' % backend_dir, '%s/public' % backend_dir],
  labels=["everest"]
)

# Build frontend
local_resource(
  'frontend-build',
  'make -C %s build EVEREST_OUT_DIR=%s/public/dist' % (frontend_dir, backend_dir),
  deps=['%s/apps' % frontend_dir],
  ignore=['%s/apps/*/dist' % frontend_dir, '%s/apps/*/.turbo' % frontend_dir],
  labels=["everest"]
)

# Live update the Everest container without generating a new pod
docker_build_with_restart('perconalab/everest',
  context='%s' % backend_dir,
  dockerfile='./everest.Dockerfile',
  entrypoint='/home/tilt/everest-api',
  only=['%s/bin/percona-everest-backend' % backend_dir],
  live_update=[
    sync('%s/bin/percona-everest-backend' % backend_dir, '/home/tilt/everest-api'),
  ]
)

k8s_yaml(namespace_inject('%s/deploy/quickstart-k8s.yaml' % backend_dir, everest_namespace))
k8s_resource(
  workload='percona-everest',
  objects=[
    'everest-admin:serviceaccount',
    'everest-admin-role:role',
    'everest-admin-role-binding:rolebinding',
    'everest-admin-cluster-role:clusterrole',
    'everest-admin-cluster-role-binding:clusterrolebinding',
    'everest-admin-token:secret',
  ],
  new_name='everest',
  port_forwards=8080,
  labels=['everest'],
)
