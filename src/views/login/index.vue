<template>
  <div class="login-container">
    <el-form ref="loginForm" :model="loginForm" :rules="loginRules" class="login-form" autocomplete="on" label-position="left" @keyup.enter.native="handleLogin()">

      <div class="title-container">
        <h3 class="title">
          {{ $t('login.title') }}
        </h3>
      </div>

      <el-form-item prop="username">
        <span class="svg-container">
          <svg-icon icon-class="user" />
        </span>
        <el-input
          ref="username"
          v-model="loginForm.username"
          :placeholder="$t('login.username')"
          name="username"
          type="text"
          tabindex="1"
          autocomplete="on"
        />
      </el-form-item>

      <el-tooltip v-model="capsTooltip" content="Caps lock is On" placement="right" manual>
        <el-form-item prop="password">
          <span class="svg-container">
            <svg-icon icon-class="password" />
          </span>
          <el-input
            :key="passwordType"
            ref="password"
            v-model="loginForm.password"
            :type="passwordType"
            :placeholder="$t('login.password')"
            name="password"
            tabindex="2"
            autocomplete="on"
            @keyup.native="checkCapslock"
            @blur="capsTooltip = false"
            @keyup.enter.native="handleLogin"
          />
          <!--<span class="show-pwd" @click="showPwd">
            <svg-icon :icon-class="passwordType === 'password' ? 'eye' : 'eye-open'" />
          </span>-->
        </el-form-item>
      </el-tooltip>
      <el-row :gutter="20">

        <el-col :span="13">
          <el-form-item prop="kaptchaCode">
            <span class="svg-container">
              <svg-icon :icon-class="'kaptcha'" />
            </span>
            <el-input ref="kaptcha" v-model="loginForm.kaptchaCode" :placeholder="$t('login.kaptchaCode')" />
          </el-form-item>
        </el-col>

        <el-col :span="10">
          <span class="show-kaptcha">
            <img :src="kaptchaBase64" @click="getKaptcha()">
          </span>
        </el-col>
      </el-row>
      <el-button :loading="loading" type="primary" style="width:100%;margin-bottom:30px;" @click.native.prevent="handleLogin">
        {{ $t('login.logIn') }}
      </el-button>
    </el-form>
  </div>
</template>

<script>
import debounce from 'lodash/debounce'
import Qs from 'qs'
export default {
  name: 'Login',
  data() {
    return {
      kaptchaBase64: '',
      loginForm: {
        grant_type: 'password',
        kaptchaUuid: '',
        username: '',
        password: '',
        kaptchaCode: ''
      },
      loginRules: {
        username: [{ required: true, trigger: 'blur', message: this.$t('form.require') }],
        password: [{ required: true, trigger: 'blur', message: this.$t('form.require') }],
        kaptchaCode: [{ required: true, trigger: 'blur', message: this.$t('form.require') }]
      },
      passwordType: 'password',
      capsTooltip: false,
      loading: false,
      showDialog: false,
      redirect: undefined,
      otherQuery: {}
    }
  },
  watch: {
    $route: {
      handler: function(route) {
        const query = route.query
        if (query) {
          this.redirect = query.redirect
          this.otherQuery = this.getOtherQuery(query)
        }
      },
      immediate: true
    }
  },
  created() {
    // window.addEventListener('storage', this.afterQRScan)
    this.getKaptcha()
  },
  mounted() {
    if (this.loginForm.username === '') {
      this.$refs.username.focus()
    } else if (this.loginForm.password === '') {
      this.$refs.password.focus()
    } else if (this.loginForm.kaptchaCode === '') {
      this.$refs.kaptcha.focus()
    }
  },
  destroyed() {
    // window.removeEventListener('storage', this.afterQRScan)
    this.getKaptcha()
  },
  methods: {
    // 获取验证码
    getKaptcha() {
      this.$http.post('/createKaptcha').then(({ data: res }) => {
        if (res.data.code && res.data.code === 200) {
          this.$message.error('验证码获取异常')
          return
        }
        this.kaptchaBase64 = 'data:image/jpeg;base64,' + res.data.img
        this.loginForm.kaptchaUuid = res.data.kaptchaUuid
      }).catch(() => {
        this.isLogin = false
      })
    },
    checkCapslock(e) {
      const { key } = e
      this.capsTooltip = key && key.length === 1 && (key >= 'A' && key <= 'Z')
    },
    showPwd() {
      if (this.passwordType === 'password') {
        this.passwordType = ''
      } else {
        this.passwordType = 'password'
      }
      this.$nextTick(() => {
        this.$refs.password.focus()
      })
    },
    handleLogin: debounce(function() {
      // chedata是我需要传递给后端的参数
      const loginParam = Qs.stringify(this.loginForm)
      this.$refs.loginForm.validate(valid => {
        if (valid) {
          this.loading = true
          this.$http.post('/oauth/token', loginParam, { headers: { 'Content-Type': 'application/x-www-form-urlencoded;charset=utf-8' }})
            .then(({ data: res }) => {
              if (res.access_token) {
                this.$router.push({ path: '/' })
              } else {
                if (res.code !== 501) {
                  this.$refs.loginForm.resetFields()
                }
                this.$message.error(res.msg)
              }
              this.getKaptcha()
              this.loading = false
            })
            .catch(() => {
              this.loading = false
            })
        } else {
          console.log('error submit!!')
          return false
        }
      })
    }, 1000, { 'leading': true, 'trailing': false }),
    getOtherQuery(query) {
      return Object.keys(query).reduce((acc, cur) => {
        if (cur !== 'redirect') {
          acc[cur] = query[cur]
        }
        return acc
      }, {})
    }
  }
}
</script>

<style lang="scss">
/* 修复input 背景不协调 和光标变色 */
/* Detail see https://github.com/PanJiaChen/vue-element-admin/pull/927 */

$bg:#283443;
$light_gray:#fff;
$cursor: #fff;

@supports (-webkit-mask: none) and (not (cater-color: $cursor)) {
  .login-container .el-input input {
    color: $cursor;
  }
}

/* reset element-ui css */
.login-container {
  .el-input {
    display: inline-block;
    height: 47px;
    width: 85%;

    input {
      background: transparent;
      border: 0px;
      -webkit-appearance: none;
      border-radius: 0px;
      padding: 12px 5px 12px 15px;
      color: $light_gray;
      height: 47px;
      caret-color: $cursor;

      &:-webkit-autofill {
        box-shadow: 0 0 0px 1000px $bg inset !important;
        -webkit-text-fill-color: $cursor !important;
      }
    }
  }

  .el-form-item {
    border: 1px solid rgba(255, 255, 255, 0.1);
    background: rgba(0, 0, 0, 0.1);
    border-radius: 5px;
    color: #454545;
  }
}
</style>

<style lang="scss" scoped>
$bg:#2d3a4b;
$dark_gray:#889aa4;
$light_gray:#eee;

.login-container {
  min-height: 100%;
  width: 100%;
  background-color: $bg;
  overflow: hidden;

  .login-form {
    position: relative;
    width: 520px;
    max-width: 100%;
    padding: 160px 35px 0;
    margin: 0 auto;
    overflow: hidden;
  }

  .tips {
    font-size: 14px;
    color: #fff;
    margin-bottom: 10px;

    span {
      &:first-of-type {
        margin-right: 16px;
      }
    }
  }

  .svg-container {
    padding: 6px 5px 6px 15px;
    color: $dark_gray;
    vertical-align: middle;
    width: 30px;
    display: inline-block;
  }

  .title-container {
    position: relative;

    .title {
      font-size: 26px;
      color: $light_gray;
      margin: 0px auto 40px auto;
      text-align: center;
      font-weight: bold;
    }

    .set-language {
      color: #fff;
      position: absolute;
      top: 3px;
      font-size: 18px;
      right: 0px;
      cursor: pointer;
    }
  }

  .show-pwd {
    position: absolute;
    right: 10px;
    top: 7px;
    font-size: 16px;
    color: $dark_gray;
    cursor: pointer;
    user-select: none;
  }
  .show-kaptcha {
    position: absolute;
    right: 10px;
    font-size: 16px;
    color: $dark_gray;
    cursor: pointer;
    user-select: none;
  }
  .thirdparty-button {
    position: absolute;
    right: 0;
    bottom: 6px;
  }

  @media only screen and (max-width: 470px) {
    .thirdparty-button {
      display: none;
    }
  }
}
</style>
