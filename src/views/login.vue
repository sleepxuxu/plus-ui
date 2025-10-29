<template>
  <div class="login">
    <div class="background">
      <div class="logo-text">
        <img src="@/assets/logo/logo.png" alt="" style="height: auto;max-width: 100%; margin-right: 10px;">
        <p style="font-size: 18px; font-weight: bold;">Duo Wan Admin</p>
      </div>
      <div class="login-background">
        <div class="background-blur"></div>
        <div class="theme-overlay">
          <img src="@/assets/logo/theme.svg" style="width: 400px; height: auto;" alt="Theme Image" />
        </div>
      </div>
      <el-form ref="loginRef" :model="loginForm" :rules="loginRules" class="login-form">
        <div class="title-box">
          <h3 class="title">{{ title }}</h3>
          <lang-select />
        </div>
        <div class="form-container">
          <el-form-item v-if="tenantEnabled" prop="tenantId">
            <el-select v-model="loginForm.tenantId" filterable :placeholder="proxy.$t('login.selectPlaceholder')"
              style="width: 100%">
              <el-option v-for="item in tenantList" :key="item.tenantId" :label="item.companyName"
                :value="item.tenantId"></el-option>
              <template #prefix><svg-icon icon-class="company" class="el-input__icon input-icon" /></template>
            </el-select>
          </el-form-item>
          <el-form-item prop="username">
            <el-input v-model="loginForm.username" type="text" size="large" auto-complete="off"
              :placeholder="proxy.$t('login.username')">
              <template #prefix><svg-icon icon-class="user" class="el-input__icon input-icon" /></template>
            </el-input>
          </el-form-item>
          <el-form-item prop="password">
            <el-input v-model="loginForm.password" type="password" size="large" auto-complete="off"
              :placeholder="proxy.$t('login.password')" @keyup.enter="handleLogin">
              <template #prefix><svg-icon icon-class="password" class="el-input__icon input-icon" /></template>
            </el-input>
          </el-form-item>
          <el-form-item v-if="captchaEnabled" prop="code">
            <el-input v-model="loginForm.code" size="large" auto-complete="off" :placeholder="proxy.$t('login.code')"
              style="width: 63%" @keyup.enter="handleLogin">
              <template #prefix><svg-icon icon-class="validCode" class="el-input__icon input-icon" /></template>
            </el-input>
            <div class="login-code">
              <img :src="codeUrl" class="login-code-img" @click="getCode" />
            </div>
          </el-form-item>
          <el-checkbox v-model="loginForm.rememberMe"
            style="margin: 0 0 25px 0">{{ proxy.$t('login.rememberPassword') }}</el-checkbox>
          <el-form-item style="float: right">
            <el-button circle :title="proxy.$t('login.social.wechat')" @click="doSocialLogin('wechat')">
              <svg-icon icon-class="wechat" />
            </el-button>
            <el-button circle :title="proxy.$t('login.social.maxkey')" @click="doSocialLogin('maxkey')">
              <svg-icon icon-class="maxkey" />
            </el-button>
            <el-button circle :title="proxy.$t('login.social.topiam')" @click="doSocialLogin('topiam')">
              <svg-icon icon-class="topiam" />
            </el-button>
            <el-button circle :title="proxy.$t('login.social.gitee')" @click="doSocialLogin('gitee')">
              <svg-icon icon-class="gitee" />
            </el-button>
            <el-button circle :title="proxy.$t('login.social.github')" @click="doSocialLogin('github')">
              <svg-icon icon-class="github" />
            </el-button>
          </el-form-item>
          <el-form-item style="width: 100%">
            <el-button :loading="loading" size="large" type="primary" style="width: 100%" @click.prevent="handleLogin">
              <span v-if="!loading">{{ proxy.$t('login.login') }}</span>
              <span v-else>{{ proxy.$t('login.logging') }}</span>
            </el-button>
            <div v-if="register" style="float: right">
              <router-link class="link-type" :to="'/register'">{{ proxy.$t('login.switchRegisterPage') }}</router-link>
            </div>
          </el-form-item>
        </div>
      </el-form>

    </div>
  </div>
</template>

<script setup lang="ts">
import { getCodeImg, getTenantList } from '@/api/login';
import { authBinding } from '@/api/system/social/auth';
import { useUserStore } from '@/store/modules/user';
import { LoginData, TenantVO } from '@/api/types';
import { to } from 'await-to-js';
import { HttpStatus } from '@/enums/RespEnum';
import { useI18n } from 'vue-i18n';

const { proxy } = getCurrentInstance() as ComponentInternalInstance;

const title = import.meta.env.VITE_APP_TITLE;
const userStore = useUserStore();
const router = useRouter();
const { t } = useI18n();

const loginForm = ref<LoginData>({
  tenantId: '000000',
  username: 'admin',
  password: 'admin123',
  rememberMe: false,
  code: '',
  uuid: ''
} as LoginData);

const loginRules: ElFormRules = {
  tenantId: [{ required: true, trigger: 'blur', message: t('login.rule.tenantId.required') }],
  username: [{ required: true, trigger: 'blur', message: t('login.rule.username.required') }],
  password: [{ required: true, trigger: 'blur', message: t('login.rule.password.required') }],
  code: [{ required: true, trigger: 'change', message: t('login.rule.code.required') }]
};

const codeUrl = ref('');
const loading = ref(false);
// 验证码开关
const captchaEnabled = ref(true);
// 租户开关
const tenantEnabled = ref(true);

// 注册开关
const register = ref(false);
const redirect = ref('/');
const loginRef = ref<ElFormInstance>();
// 租户列表
const tenantList = ref<TenantVO[]>([]);

watch(
  () => router.currentRoute.value,
  (newRoute: any) => {
    redirect.value = newRoute.query && newRoute.query.redirect && decodeURIComponent(newRoute.query.redirect);
  },
  { immediate: true }
);

const handleLogin = () => {
  loginRef.value?.validate(async (valid: boolean, fields: any) => {
    if (valid) {
      loading.value = true;
      // 勾选了需要记住密码设置在 localStorage 中设置记住用户名和密码
      if (loginForm.value.rememberMe) {
        localStorage.setItem('tenantId', String(loginForm.value.tenantId));
        localStorage.setItem('username', String(loginForm.value.username));
        localStorage.setItem('password', String(loginForm.value.password));
        localStorage.setItem('rememberMe', String(loginForm.value.rememberMe));
      } else {
        // 否则移除
        localStorage.removeItem('tenantId');
        localStorage.removeItem('username');
        localStorage.removeItem('password');
        localStorage.removeItem('rememberMe');
      }
      // 调用action的登录方法
      const [err] = await to(userStore.login(loginForm.value));
      if (!err) {
        const redirectUrl = redirect.value || '/';
        await router.push(redirectUrl);
        loading.value = false;
      } else {
        loading.value = false;
        // 重新获取验证码
        if (captchaEnabled.value) {
          await getCode();
        }
      }
    } else {
      console.log('error submit!', fields);
    }
  });
};

/**
 * 获取验证码
 */
const getCode = async () => {
  const res = await getCodeImg();
  const { data } = res;
  captchaEnabled.value = data.captchaEnabled === undefined ? true : data.captchaEnabled;
  if (captchaEnabled.value) {
    codeUrl.value = 'data:image/gif;base64,' + data.img;
    loginForm.value.uuid = data.uuid;
  }
};

const getLoginData = () => {
  const tenantId = localStorage.getItem('tenantId');
  const username = localStorage.getItem('username');
  const password = localStorage.getItem('password');
  const rememberMe = localStorage.getItem('rememberMe');
  loginForm.value = {
    tenantId: tenantId === null ? String(loginForm.value.tenantId) : tenantId,
    username: username === null ? String(loginForm.value.username) : username,
    password: password === null ? String(loginForm.value.password) : String(password),
    rememberMe: rememberMe === null ? false : Boolean(rememberMe)
  } as LoginData;
};

/**
 * 获取租户列表
 */
const initTenantList = async () => {
  const { data } = await getTenantList(false);
  tenantEnabled.value = data.tenantEnabled === undefined ? true : data.tenantEnabled;
  if (tenantEnabled.value) {
    tenantList.value = data.voList;
    if (tenantList.value != null && tenantList.value.length !== 0) {
      loginForm.value.tenantId = tenantList.value[0].tenantId;
    }
  }
};

/**
 * 第三方登录
 * @param type
 */
const doSocialLogin = (type: string) => {
  authBinding(type, loginForm.value.tenantId).then((res: any) => {
    if (res.code === HttpStatus.SUCCESS) {
      // 获取授权地址跳转
      window.location.href = res.data;
    } else {
      ElMessage.error(res.msg);
    }
  });
};

onMounted(() => {
  getCode();
  initTenantList();
  getLoginData();
});
</script>

<style lang="scss" scoped>
.login {
  height: 100%;
  width: 100%;

  .background {
    display: flex;
    background-color: #f1f3f6;
    height: 100%;
    position: relative;

    .logo-text {
      display: flex;
      position: absolute;
      justify-content: center;
      align-items: center;
      margin-left: 18px;
      margin-top: 18px;
      z-index: 10;
    }

    .login-background {
      flex: 1 1 0%;
      position: relative;
      overflow: hidden;
    }

    .background-blur {
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background: linear-gradient(154deg, #07070915 30%, #006be64d 48%, #07070915 64%);
      filter: blur(100px);
      z-index: 1;
    }

    .theme-overlay {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      z-index: 5;
      animation: float-animation 3s ease-in-out infinite;
    }
  }
}

.title-box {
  display: flex;
  margin-top: 40px;

  .title {
    margin: 0px auto 30px auto;
    text-align: center;
    color: #707070;
  }

  :deep(.lang-select--style) {
    line-height: 0;
    color: #7483a3;
  }
}

.login-form {
  background: #ffffff;
  width: 34%;
  padding: 25px 25px 5px 25px;
  z-index: 20;
  flex: 0 1 auto;
  position: relative;

  .el-input {
    height: 40px;

    input {
      height: 40px;
    }
  }

  .input-icon {
    height: 39px;
    width: 14px;
    margin-left: 0px;
  }
}

.form-container {
  margin-top: 100px;
  padding: 50px;
}

.login-tip {
  font-size: 13px;
  text-align: center;
  color: #bfbfbf;
}

.login-code {
  width: 33%;
  height: 40px;
  float: right;

  img {
    cursor: pointer;
    vertical-align: middle;
  }
}

.login-code-img {
  height: 40px;
  padding-left: 12px;
}

@keyframes float-animation {
  0% {
    transform: translate(-50%, -50%) translateY(0px);
  }

  50% {
    transform: translate(-50%, -50%) translateY(-10px);
  }

  100% {
    transform: translate(-50%, -50%) translateY(0px);
  }
}
</style>
