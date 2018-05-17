# Gopkg.toml in the wild

## Starting set

Starting with all of the packages in the [corpus][corpus], I first filtered to only be the packages with an import path starting with github.com. That left [509][githubs] packages. Out of those packages, [33][withdep] had a `Gopkg.toml` in their root on the master branch.

## Results

### Types of constraints

Out of the [323][constraints] constraints identified, the breakdown of the type is

| Kind     | Count | Percent |
| ----     | ----- | ------- |
| Branch   | 139   | 43%     |
| Revision | 29    | 9%      |
| Version  | 155   | 48%     |

The vast majority of the branch constraints (127 / 139 = 91%) are against master. This is the same thing as you would get with vanilla `go get`.

### Revision Constraints

Most (25 / 29 = 86%) of the revision constraints are in binary packages: things that are at the top of the build dependency in every situation. In those cases, pinning isn't imporant. Of the remainder, only one is potentially valid to allow building on an older version of Go.

| Import path            | Depends on                      | Revision   | Determination           |
| -----------            | ----------                      | --------   | -------------           |
| aws/aws-sdk-go         | github.com/jmespath/go-jmespath | `0b12d6b5` | unnecessary<sup>1</sup> |
| fsouza/go-dockerclient | github.com/docker/docker        | `a422774e` | valid<sup>2</sup>       |
| fsouza/go-dockerclient | github.com/ijc25/Gotty          | `a8b993ba` | unnecessary<sup>3</sup> |
| nsqio/go               | github.com/golang/snappy        | `d9eb7a3d` | unnecessary<sup>4</sup> |

<sup>1</sup> Between the latest release `0.2.2` and the pin of `0b12d6b5`, only minor gofmt changes happened.

<sup>2</sup> This is a library, but the pin is only required to continue to build against go1.9. It's unclear if some other version would be fine.

<sup>3</sup> The pinned version is the most recent master.

<sup>4</sup> This pinned version has no significant commits up to master.

### Version Constraints

| Kind  | Count | Percent |
| ----  | ----- | ------- |
| ^     | 130   | 84%     |
| ~     | 10    | 6%      |
| =     | 8     | 5%      |
| Other | 7     | 5%      |

The `~` style constraints are more restricted in that it's anything in the same minor version, so that `~1.2` is the same as `>=1.2, <1.3`. Since that's close enough in meaning to `^`, I did not bother to do an analysis to see if it was required. Of the `=` constraints, they were all in a binary. In addition, one was only a pin because upstream had a pin that it had to deal with.

The other constraints were very strange and all in gravitational/teleport.

| Constraint                   | Depends on                          | Determination                   |
| ----------                   | ----------                          | -------------                   |
| `>=0.0.2, <=1.0.0-gc98f59f`  | github.com/mailgun/lemma            | there is no version 1           |
| `>=0.3.1, <=12.0.0-gccfb39d` | github.com/google/gops              | there is no version 1           |
| `>=1.0.0, <=11.0.0-gc55201b` | github.com/pborman/uuid             | there is no version 11          |
| `>=1.0.0, <=111.0.0-g04a3e8` | github.com/boltdb/bolt              | there is no version 111         |
| `>=1.0.0, <=183.0.0-g231b4c` | google.golang.org/grpc              | there is no version 183         |
| `>=1.0.0, <=19.0.0-g5d150e7` | github.com/cenkalti/backoff         | there is no version 19, just 2. |
| `>=1.0.0, <=64.0.0-gb428fda` | github.com/julienschmidt/httprouter | there is no version 64          |

This makes all of these constraints defacto `^`, bringing the total of non `^`ish constraints to 147 / 155 = 95%. Of the remainder 5%, they were all in binaries.

### Version selection

Just because all of the constraints are `^` does not mean they would select the same versions as vgo. Perhaps some conflict would require them to select something that was not the newest version, which would cause problems. For every package that had a .lock file published, I checked if the selected version is currently the maximal version for that dependency. In every case, the maximal version at the time was selected. And in the majority of cases, that version is still the maximal version.

| Import path                   | Depends on                             | Constraint                    | Selected  | Determination        |
| -----------                   | ----------                             | ----------                    | --------  | -------------        |
| Azure/azure-sdk-for-go        | github.com/shopspring/decimal          | `1.0.0`                       | `1.0.1`   | maximal              |
| bitly/oauth2_proxy            | gopkg.in/fsnotify.v1                   | `~1.2.0`                      | `1.2.11`  | maximal (in 1.2)     |
| gravitational/teleport        | github.com/aws/aws-sdk-go              | `1.12.17`                     | `1.12.25` | 1.13.47.<sup>1</sup> |
| gravitational/teleport        | github.com/boltdb/bolt                 | `>=1.0.0, <=111.0.0-g04a3e85` | `1.3.1`   | maximal              |
| gravitational/teleport        | github.com/coreos/etcd                 | `3.2.4`                       | `3.2.6`   | 3.3.5.<sup>2</sup>   |
| gravitational/teleport        | github.com/julienschmidt/httprouter    | `>=1.0.0, <=64.0.0-gb428fda`  | `1.1`     | maximal              |
| gravitational/teleport        | github.com/pborman/uuid                | `>=1.0.0, <=11.0.0-gc55201b`  | `1.1`     | maximal              |
| gravitational/teleport        | google.golang.org/grpc                 | `>=1.0.0, <=183.0.0-g231b4cf` | `1.5.2`   | 1.12.0.<sup>3</sup>  |
| hyperledger/fabric            | github.com/fsouza/go-dockerclient      | `1.0.0`                       | `1.2.0`   | maximal              |
| kelseyhightower/confd         | github.com/sirupsen/logrus             | `1.0.3`                       | `1.0.5`   | maximal              |
| kelseyhightower/confd         | github.com/aws/aws-sdk-go              | `1.12.4`                      | `1.13.41` | 1.13.47.<sup>4</sup> |
| kelseyhightower/confd         | github.com/fsnotify/fsnotify           | `1.4.2`                       | `1.4.7`   | maximal              |
| kelseyhightower/confd         | github.com/garyburd/redigo             | `1.1.0`                       | `1.6.0`   | maximal              |
| lightstep/lightstep-tracer-go | google.golang.org/grpc                 | `1.4.3`                       | `1.8.1`   | 1.12.0.<sup>5</sup>  |
| maruel/panicparse             | github.com/mattn/go-isatty             | `0.0.2`                       | `0.0.3`   | maximal              |
| pingcap/pd                    | github.com/grpc-ecosystem/grpc-gateway | `~1.3`                        | `1.3.1`   | maximal              |
| pingcap/tidb                  | github.com/opentracing/opentracing-go  | `1.0.0`                       | `1.0.2`   | maximal              |
| spf13/hugo                    | github.com/magefile/mage               | `v1`                          | `1.0.2`   | maximal (in v1)      |
| spf13/hugo                    | github.com/fortytw2/leaktest           | `1.1.0`                       | `1.2.0`   | maximal              |
| spf13/hugo                    | github.com/fsnotify/fsnotify           | `1.4.0`                       | `1.4.7`   | maximal              |
| spf13/hugo                    | github.com/spf13/cast                  | `^1.1.0`                      | `1.2.0`   | maximal              |
| spf13/hugo                    | github.com/spf13/cobra                 | `^0.0.1`                      | `0.0.2`   | maximal              |
| spf13/hugo                    | github.com/spf13/viper                 | `1.0.0`                       | `1.0.2`   | maximal              |
| spf13/hugo                    | github.com/stretchr/testify            | `1.1.4`                       | `1.2.1`   | maximal              |
| spf13/hugo                    | github.com/gobwas/glob                 | `0.0.2`                       | `0.2.3`   | maximal              |

<sup>1</sup> at the time [1.12.25](https://github.com/aws/aws-sdk-go/commit/a201bf33b18ad4ab54344e4bc26b87eb6ad37b8e) was maximal when [the commit](https://github.com/gravitational/teleport/commit/cd2d2726de5f2dd6ffb1d375c1a3bd2739f04894) added it to the lock.

<sup>2</sup> at the time [3.2.6](https://github.com/coreos/etcd/tree/v3.2.6) was maximal when [the commit](https://github.com/gravitational/teleport/commit/8b81a0c3841845a0e4399301c2ac47f23f3a676f) added it to the lock.


<sup>3</sup> at the time [v1.5.2](https://github.com/grpc/grpc-go/tree/v1.5.2) was maximal when [the commit](https://github.com/gravitational/teleport/commit/8b81a0c3841845a0e4399301c2ac47f23f3a676f) added it to the lock.


<sup>4</sup> at the time [v1.13.41](https://github.com/aws/aws-sdk-go/tree/v1.13.41) was maximal (or within a day of maximal) when [the commit](https://github.com/kelseyhightower/confd/commit/d73ef600a2e3d4f779b27b6f52364481c0a0a37c) added it to the lock. aws-sdk-go moves astonishingly quickly, releasing minor versions every couple of days.

<sup>5</sup> 1.8.1 no longer exists. it goes from 1.8.0 to 1.8.2, suggesting that this version is broken. the commit it was at though is [here](https://github.com/grpc/grpc-go/commit/be077907e29fdb945d351e4284eb5361e7f8924e) which is only 4 days before when [this commit](https://github.com/lightstep/lightstep-tracer-go/commit/c9dcbe57aee3cbcf0e08966847369601e169ad30) introduced it to the lock file, suggesting it is likely maximal.

# Summary

While the sample size is small, it's not nothing. It also includes a number of high profile packages in the community.

- Almost half of all constraints aren't actually constraints at all: they point at the master branch.
- Of the constraints that do select a version, they are almost always of the >= within the same major version type.
- Of the constraints that aren't of that type, every one of them is pinning to a specific version.
- Of the pins, only one is not in a binary with a possibly legitimate reason.
- When the version constraints are resolved, there is no evidence of any kind of conflict, and trivially picking the largest version at the time always worked.

I encourage everyone to attempt to find cases in the wild where dependency resolution had problems, so that we can make good experience reports, and get a better understanding of the theoretical vgo world. All this took was grep, curl, and some elbow grease. You can do it too! Thank you.

[corpus]: https://github.com/rsc/corpus
[githubs]: https://github.com/zeebo/dep-analysis/blob/master/githubs
[withdep]: https://github.com/zeebo/dep-analysis/blob/master/withdep
[constraints]: https://github.com/zeebo/dep-analysis/blob/master/constraints
